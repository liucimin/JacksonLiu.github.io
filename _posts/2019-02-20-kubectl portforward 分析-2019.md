---
layout:     post                    # 使用的布局（不需要改）
title:      Kubernets源码分析             # 标题 
subtitle:    kubectl port-forward发生了什么
date:       2019-03-13             # 时间
author:     Jackson Liu                      # 作者
header-img: img/post-bg-mma-0.png    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Kubernets
---



### 一、kubectl方面：

#### 1、命令初始化时，使用了cobra库来初始化，cobra库具体可以看另一篇的分析。




```c
func NewCmdPortForward(f cmdutil.Factory, streams genericclioptions.IOStreams) *cobra.Command {
	opts := &PortForwardOptions{
		PortForwarder: &defaultPortForwarder{
			IOStreams: streams,
		},
	}
	cmd := &cobra.Command{
		Use:                   "port-forward TYPE/NAME [options][LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N]",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Forward one or more local ports to a pod"),
		Long:                  portforwardLong,
		Example:               portforwardExample,
		Run: func(cmd *cobra.Command, args []string) {
			if err := opts.Complete(f, cmd, args); err != nil {
				cmdutil.CheckErr(err)
			}
			if err := opts.Validate(); err != nil {
				cmdutil.CheckErr(cmdutil.UsageErrorf(cmd, "%v", err.Error()))
			}
			if err := opts.RunPortForward(); err != nil {
				cmdutil.CheckErr(err)
			}
		},
	}
	cmdutil.AddPodRunningTimeoutFlag(cmd, defaultPodPortForwardWaitTimeout)
	cmd.Flags().StringSliceVar(&opts.Address, "address", []string{"localhost"}, "Addresses to listen on (comma separated). Only accepts IP addresses or localhost as a value. When localhost is supplied, kubectl will try to bind on both 127.0.0.1 and ::1 and will fail if neither of these addresses are available to bind.")
	// TODO support UID
	return cmd
}
```





#### 2、当执行 kubectl port-forward nginx-pod2 6379:6379 类似命令时，会进入kubectl的如下流程：




```c

// RunPortForward implements all the necessary functionality for port-forward cmd.
func (o PortForwardOptions) RunPortForward() error {
    
	pod, err := o.PodClient.Pods(o.Namespace).Get(o.PodName, metav1.GetOptions{})
	if err != nil {
		return err
	}

	if pod.Status.Phase != corev1.PodRunning {
		return fmt.Errorf("unable to forward port because pod is not running. Current status=%v", pod.Status.Phase)
	}

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)
	defer signal.Stop(signals)

	go func() {
		<-signals
		if o.StopChannel != nil {
			close(o.StopChannel)
		}
	}()

	req := o.RESTClient.Post().
		Resource("pods").
		Namespace(o.Namespace).
		Name(pod.Name).
		SubResource("portforward")

	return o.PortForwarder.ForwardPorts("POST", req.URL(), o)
}

```



RunPortForward 主要流程：

（1）查询pod是否running状态

（2）注册系统消息os.Interrupt，当出现时，关闭ForwardPorts

（3）构造一个k8s client的resetful request（portforward）请求，这里使用的是POST请求，后面会分析为什么不是HTTP的CONNECT请求。

 /api/v1/namespaces/{namespace}/pods/{name}/portforward 

（4）调用ForwardPorts

```c
func (f *defaultPortForwarder) ForwardPorts(method string, url *url.URL, opts PortForwardOptions) error {

	transport, upgrader, err := spdy.RoundTripperFor(opts.Config)

	if err != nil {

		return err

	}

	dialer := spdy.NewDialer(upgrader, &http.Client{Transport: transport}, method, url)

	fw, err := portforward.NewOnAddresses(dialer, opts.Address, opts.Ports, opts.StopChannel, opts.ReadyChannel, f.Out, f.ErrOut)

	if err != nil {

		return err

	}

	return fw.ForwardPorts()

}

```




defaultPortForwarder的ForwardPorts函数主要流程：

(1)生成使用*SPDY*协议的 round tripper and upgrader，

*SPDY*（读作“SPeeDY”）是Google开发的基于TCP的应用层协议，用以最小化网络延迟，提升网络速度，优化用户的网络使用体验。 

*SPDY*也就是HTTP/2的前身 。

（2）生成PortForwarder，用于正在的port forward

（3）启动 PortForwarder.ForwardPorts()

```c
// ForwardPorts formats and executes a port forwarding request. The connection will remain
// open until stopChan is closed.
func (pf *PortForwarder) ForwardPorts() error {
	defer pf.Close()

	var err error
	pf.streamConn, _, err = pf.dialer.Dial(PortForwardProtocolV1Name)
	if err != nil {
		return fmt.Errorf("error upgrading connection: %s", err)
	}
	defer pf.streamConn.Close()

	return pf.forward()
}

// forward dials the remote host specific in req, upgrades the request, starts
// listeners for each port specified in ports, and forwards local connections
// to the remote host via streams.
func (pf *PortForwarder) forward() error {
	var err error

	listenSuccess := false
	for _, port := range pf.ports {
		err = pf.listenOnPort(&port)
		switch {
		case err == nil:
			listenSuccess = true
		default:
			if pf.errOut != nil {
				fmt.Fprintf(pf.errOut, "Unable to listen on port %d: %v\n", port.Local, err)
			}
		}
	}

	if !listenSuccess {
		return fmt.Errorf("Unable to listen on any of the requested ports: %v", pf.ports)
	}

	if pf.Ready != nil {
		close(pf.Ready)
	}

	// wait for interrupt or conn closure
	select {
	case <-pf.stopChan:
	case <-pf.streamConn.CloseChan():
		runtime.HandleError(errors.New("lost connection to pod"))
	}

	return nil
}

```

大概说一下这里的思路，

（1）创建一个spdy stream准备和api-server交互，

其中spdy stream是一个多路复用的tcp链接，所以后面会看到在这个stream上可以生成 error steam 和data steam。

（2）根据传参，监听对应的local port。即kubectl这个进程监听需要转发的port。



#### 3、这里首先聊聊kubectl监听端口后的操作：

对于每个local address 和local port会启动listener，并且阻塞在accept。

```c
// waitForConnection waits for new connections to listener and handles them in
// the background.
func (pf *PortForwarder) waitForConnection(listener net.Listener, port ForwardedPort) {
	for {
		conn, err := listener.Accept()
		if err != nil {
			// TODO consider using something like https://github.com/hydrogen18/stoppableListener?
			if !strings.Contains(strings.ToLower(err.Error()), "use of closed network connection") {
				runtime.HandleError(fmt.Errorf("Error accepting connection on port %d: %v", port.Local, err))
			}
			return
		}
		go pf.handleConnection(conn, port)
	}
}

```



当在本地访问对应端口的时候会走到handleConnection



```c
// handleConnection copies data between the local connection and the stream to

// the remote server.

func (pf *PortForwarder) handleConnection(conn net.Conn, port ForwardedPort) {

	defer conn.Close()

    if pf.out != nil {
        fmt.Fprintf(pf.out, "Handling connection for %d\n", port.Local)
    }

    requestID := pf.nextRequestID()

    // create error stream
    headers := http.Header{}
    headers.Set(v1.StreamType, v1.StreamTypeError)
    headers.Set(v1.PortHeader, fmt.Sprintf("%d", port.Remote))
    headers.Set(v1.PortForwardRequestIDHeader, strconv.Itoa(requestID))
    errorStream, err := pf.streamConn.CreateStream(headers)
    if err != nil {
        runtime.HandleError(fmt.Errorf("error creating error stream for port %d -> %d: %v", port.Local, port.Remote, err))
        return
    }
    // we're not writing to this stream
    errorStream.Close()

    errorChan := make(chan error)
    go func() {
        message, err := ioutil.ReadAll(errorStream)
        switch {
        case err != nil:
            errorChan <- fmt.Errorf("error reading from error stream for port %d -> %d: %v", port.Local, port.Remote, err)
        case len(message) > 0:
            errorChan <- fmt.Errorf("an error occurred forwarding %d -> %d: %v", port.Local, port.Remote, string(message))
        }
        close(errorChan)
    }()

    // create data stream
    headers.Set(v1.StreamType, v1.StreamTypeData)
    dataStream, err := pf.streamConn.CreateStream(headers)
    if err != nil {
        runtime.HandleError(fmt.Errorf("error creating forwarding stream for port %d -> %d: %v", port.Local, port.Remote, err))
        return
    }

    localError := make(chan struct{})
    remoteDone := make(chan struct{})

    go func() {
        // Copy from the remote side to the local port.
        if _, err := io.Copy(conn, dataStream); err != nil && !strings.Contains(err.Error(), "use of closed network connection") {
            runtime.HandleError(fmt.Errorf("error copying from remote stream to local connection: %v", err))
        }

        // inform the select below that the remote copy is done
        close(remoteDone)
    }()

    go func() {
        // inform server we're not sending any more data after copy unblocks
        defer dataStream.Close()

        // Copy from the local port to the remote side.
        if _, err := io.Copy(dataStream, conn); err != nil && !strings.Contains(err.Error(), "use of closed network connection") {
            runtime.HandleError(fmt.Errorf("error copying from local connection to remote stream: %v", err))
            // break out of the select below without waiting for the other copy to finish
            close(localError)
        }
    }()

    // wait for either a local->remote error or for copying from remote->local to finish
    select {
    case <-remoteDone:
    case <-localError:
    }

    // always expect something on errorChan (it may be nil)
    err = <-errorChan
    if err != nil {
        runtime.HandleError(err)
    }
}
```
解析一下handleConnection函数：

local port收到tcp建链后的处理过程：

1） create error stream 创建error stream

2）create data stream 创建data stream

  data stream 是与api server通信的通道。





### 二、api-server这边:

#### 1、首先分析restful接口初始化：

 /api/v1/namespaces/{namespace}/pods/{name}/portforward 

```c

//这个是route之后的controller，真正处理portforward请求的
//初始化podStorage.PortForward
PortForward: &podrest.PortForwardREST{Store: store, KubeletConn: k},

//初始化初始化podStorage
podStorage := podstore.NewStorage(
	restOptionsGetter,
	nodeStorage.KubeletConnectionInfo,
	c.ProxyTransport,
	podDisruptionClient,
)



//生成StorageMap
restStorageMap := map[string]rest.Storage{
	"pods":             podStorage.Pod,
	"pods/attach":      podStorage.Attach,
	"pods/status":      podStorage.Status,
	"pods/log":         podStorage.Log,
	"pods/exec":        podStorage.Exec,
	"pods/portforward": podStorage.PortForward,
	"pods/proxy":       podStorage.Proxy,
	"pods/binding":     podStorage.Binding,
	"bindings":         podStorage.Binding,

	"podTemplates": podTemplateStorage,

	"replicationControllers":        controllerStorage.Controller,
	"replicationControllers/status": controllerStorage.Status,

	"services":        serviceRest,
	"services/proxy":  serviceRestProxy,
	"services/status": serviceStatusStorage,

	"endpoints": endpointsStorage,

	"nodes":        nodeStorage.Node,
	"nodes/status": nodeStorage.Status,
	"nodes/proxy":  nodeStorage.Proxy,

	"events": eventStorage,

	"limitRanges":                   limitRangeStorage,
	"resourceQuotas":                resourceQuotaStorage,
	"resourceQuotas/status":         resourceQuotaStatusStorage,
	"namespaces":                    namespaceStorage,
	"namespaces/status":             namespaceStatusStorage,
	"namespaces/finalize":           namespaceFinalizeStorage,
	"secrets":                       secretStorage,
	"serviceAccounts":               serviceAccountStorage,
	"persistentVolumes":             persistentVolumeStorage,
	"persistentVolumes/status":      persistentVolumeStatusStorage,
	"persistentVolumeClaims":        persistentVolumeClaimStorage,
	"persistentVolumeClaims/status": persistentVolumeClaimStatusStorage,
	"configMaps":                    configMapStorage,

	"componentStatuses": componentstatus.NewStorage(componentStatusStorage{c.StorageFactory}.serversToValidate),
}


	apiGroupInfo.VersionedResourcesStorageMap["v1"] = restStorageMap



//在router上注册所有的http method和path
//初始化rest接口时，会判断是否connecter，从而只处理connect请求
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {
    ...
    connecter, isConnecter := storage.(rest.Connecter)
    ...
    actions = appendIf(actions, action{"CONNECT", itemPath, nameParams, namer, false}, isConnecter)
    ...    
    		
    //connect的时候的route初始化
    		case "CONNECT":
    //根据connecter.ConnectMethods()可以读到是GET,POST的方法，所以在前面可以看到使用的是POST请求，而不是connect请求。
			for _, method := range connecter.ConnectMethods() {
				connectProducedObject := storageMeta.ProducesObject(method)
				if connectProducedObject == nil {
					connectProducedObject = "string"
				}
				doc := "connect " + method + " requests to " + kind
				if isSubresource {
					doc = "connect " + method + " requests to " + subresource + " of " + kind
				}
				handler := metrics.InstrumentRouteFunc(action.Verb, resource, subresource, requestScope, restfulConnectResource(connecter, reqScope, admit, path, isSubresource))
				route := ws.Method(method).Path(action.Path).
					To(handler).
					Doc(doc).
					Operation("connect" + strings.Title(strings.ToLower(method)) + namespaced + kind + strings.Title(subresource) + operationSuffix).
					Produces("*/*").
					Consumes("*/*").
					Writes(connectProducedObject)
				if versionedConnectOptions != nil {
					if err := addObjectParams(ws, route, versionedConnectOptions); err != nil {
						return nil, err
					}
				}
				addParams(route, action.Params)
				routes = append(routes, route)

				// transform ConnectMethods to kube verbs
				if kubeVerb, found := toDiscoveryKubeVerb[method]; found {
					if len(kubeVerb) != 0 {
						kubeVerbs[kubeVerb] = struct{}{}
					}
				}
			}
    
    
}

```


这里可以看到初始化了handler：

使用restfulConnectResource来生成func。

handler := metrics.InstrumentRouteFunc(action.Verb, resource, subresource, requestScope, restfulConnectResource(connecter, reqScope, admit, path, isSubresource))

```c
func restfulConnectResource(connecter rest.Connecter, scope handlers.RequestScope, admit admission.Interface, restPath string, isSubresource bool) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.ConnectResource(connecter, scope, admit, restPath, isSubresource)(res.ResponseWriter, req.Request)
	}
}


// ConnectResource returns a function that handles a connect request on a rest.Storage object.

func ConnectResource(connecter rest.Connecter, scope RequestScope, admit admission.Interface, restPath string, isSubresource bool) http.HandlerFunc {

	return func(w http.ResponseWriter, req *http.Request) {

		if isDryRun(req.URL) {

			scope.err(errors.NewBadRequest("dryRun is not supported"), w, req)

			return

		}


    namespace, name, err := scope.Namer.Name(req)
	if err != nil {
		scope.err(err, w, req)
		return
	}
	ctx := req.Context()
	ctx = request.WithNamespace(ctx, namespace)
	ae := request.AuditEventFrom(ctx)
	admit = admission.WithAudit(admit, ae)

	opts, subpath, subpathKey := connecter.NewConnectOptions()
	if err := getRequestOptions(req, scope, opts, subpath, subpathKey, isSubresource); err != nil {
		err = errors.NewBadRequest(err.Error())
		scope.err(err, w, req)
		return
	}
	if admit != nil && admit.Handles(admission.Connect) {
		userInfo, _ := request.UserFrom(ctx)
		// TODO: remove the mutating admission here as soon as we have ported all plugin that handle CONNECT
		if mutatingAdmission, ok := admit.(admission.MutationInterface); ok {
			err = mutatingAdmission.Admit(admission.NewAttributesRecord(opts, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Connect, false, userInfo))
			if err != nil {
				scope.err(err, w, req)
				return
			}
		}
		if validatingAdmission, ok := admit.(admission.ValidationInterface); ok {
			err = validatingAdmission.Validate(admission.NewAttributesRecord(opts, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Connect, false, userInfo))
			if err != nil {
				scope.err(err, w, req)
				return
			}
		}
	}
	requestInfo, _ := request.RequestInfoFrom(ctx)
	metrics.RecordLongRunning(req, requestInfo, func() {
		handler, err := connecter.Connect(ctx, name, opts, &responder{scope: scope, req: req, w: w})
		if err != nil {
			scope.err(err, w, req)
			return
		}
		handler.ServeHTTP(w, req)
	})
}
}
```


#### 2、下面解析一下具体api-server收到post的portforward消息后的处理：

```c
// Connect returns a handler for the pod portforward proxy
func (r *PortForwardREST) Connect(ctx context.Context, name string, opts runtime.Object, responder rest.Responder) (http.Handler, error) {
	portForwardOpts, ok := opts.(*api.PodPortForwardOptions)
	if !ok {
		return nil, fmt.Errorf("invalid options object: %#v", opts)
	}
	location, transport, err := pod.PortForwardLocation(r.Store, r.KubeletConn, ctx, name, portForwardOpts)
	if err != nil {
		return nil, err
	}
	return newThrottledUpgradeAwareProxyHandler(location, transport, false, true, true, responder), nil
}
	
```

主要分为两步：

（1）获取对应pod所在的node，

获取node上kubelet的  kubelet-reported address 和 kubelet-reported port，准备连接到node的kubelet来执行从kubectl开启portford的收到的http消息。

（2）生成一个proxy handler，用来代理后面发送的data stream到被代理的服务器，这里是node上的kubelet。



api-sever这边并没有使用spdy,因为这里只做了一个中转，把所有接收到kubectl的http请求都代理到了kubelet，注意这里是http代理不是tcp代理，所以https的也能代理成功，http2.0暂时不知道能否成功。



三、kubelet方面

#### 1、先讲server的初始化，

目前看只有runtime是docker的时候才会初始化这个server端，并且是dockershim里面初始化的(因为 !crOptions.RedirectContainerStreaming， crOptions.RedirectContainerStreaming默认应该是false，然后相当于通知ds 默认开启startLocalStreamingServer)：

```c



	ds, err := dockershim.NewDockerService(kubeDeps.DockerClientConfig, crOptions.PodSandboxImage, streamingConfig,&pluginSettings, runtimeCgroups, kubeCfg.CgroupDriver, crOptions.DockershimRootDirectory, !crOptions.RedirectContainerStreaming)

	ds := &dockerService{
		client:          c,
		os:              kubecontainer.RealOS{},
		podSandboxImage: podSandboxImage,
		streamingRuntime: &streamingRuntime{
			client:      client,
			execHandler: &NativeExecHandler{},
		},
		containerManager:          cm.NewContainerManager(cgroupsName, client),
		checkpointManager:         checkpointManager,
		startLocalStreamingServer: startLocalStreamingServer,
		networkReady:              make(map[string]bool),
	}

	// create streaming server if configured.
	if streamingConfig != nil {
		var err error
		ds.streamingServer, err = streaming.NewServer(*streamingConfig, ds.streamingRuntime)
		if err != nil {
			return nil, err
		}
	}



// TODO(tallclair): Add auth(n/z) interface & handling.
func NewServer(config Config, runtime Runtime) (Server, error) {
	s := &server{
		config:  config,
		runtime: &criAdapter{runtime},
		cache:   newRequestCache(),
	}

	if s.config.BaseURL == nil {
		s.config.BaseURL = &url.URL{
			Scheme: "http",
			Host:   s.config.Addr,
		}
		if s.config.TLSConfig != nil {
			s.config.BaseURL.Scheme = "https"
		}
	}

	ws := &restful.WebService{}
	endpoints := []struct {
		path    string
		handler restful.RouteFunction
	}{
		{"/exec/{token}", s.serveExec},
		{"/attach/{token}", s.serveAttach},
		{"/portforward/{token}", s.servePortForward},
	}
	// If serving relative to a base path, set that here.
	pathPrefix := path.Dir(s.config.BaseURL.Path)
	for _, e := range endpoints {
		for _, method := range []string{"GET", "POST"} {
			ws.Route(ws.
				Method(method).
				Path(path.Join(pathPrefix, e.path)).
				To(e.handler))
		}
	}
	handler := restful.NewContainer()
	handler.Add(ws)
	s.handler = handler
	s.server = &http.Server{
		Addr:      s.config.Addr,
		Handler:   s.handler,
		TLSConfig: s.config.TLSConfig,
	}

	return s, nil
}




```
初始化比较简单，就如上面的代码流程所示。

#### 2、再讲接收到api-server转发过来的消息后的处理流程

/pkg/kubelet/server/portforward/portforward.go

```c


//收到消息后的server处理入口

func (s *server) servePortForward(req *restful.Request, resp *restful.Response) {
	token := req.PathParameter("token")
	cachedRequest, ok := s.cache.Consume(token)
	if !ok {
		http.NotFound(resp.ResponseWriter, req.Request)
		return
	}
	pf, ok := cachedRequest.(*runtimeapi.PortForwardRequest)
	if !ok {
		http.NotFound(resp.ResponseWriter, req.Request)
		return
	}

	portForwardOptions, err := portforward.BuildV4Options(pf.Port)
	if err != nil {
		resp.WriteError(http.StatusBadRequest, err)
		return
	}

	portforward.ServePortForward(
		resp.ResponseWriter,
		req.Request,
		s.runtime,
		pf.PodSandboxId,
		"", // unused: podUID
		portForwardOptions,
		s.config.StreamIdleTimeout,
		s.config.StreamCreationTimeout,
		s.config.SupportedPortForwardProtocols)
}


//继续往下跟踪
// ServePortForward handles a port forwarding request.  A single request is
// kept alive as long as the client is still alive and the connection has not
// been timed out due to idleness. This function handles multiple forwarded
// connections; i.e., multiple `curl http://localhost:8888/` requests will be
// handled by a single invocation of ServePortForward.
func ServePortForward(w http.ResponseWriter, req *http.Request, portForwarder PortForwarder, podName string, uid types.UID, portForwardOptions *V4Options, idleTimeout time.Duration, streamCreationTimeout time.Duration, supportedProtocols []string) {
	var err error
	if wsstream.IsWebSocketRequest(req) {
		err = handleWebSocketStreams(req, w, portForwarder, podName, uid, portForwardOptions, supportedProtocols, idleTimeout, streamCreationTimeout)
	} else {
		err = handleHttpStreams(req, w, portForwarder, podName, uid, supportedProtocols, idleTimeout, streamCreationTimeout)
	}

	if err != nil {
		runtime.HandleError(err)
		return
	}
}




func handleHttpStreams(req *http.Request, w http.ResponseWriter, portForwarder PortForwarder, podName string, uid types.UID, supportedPortForwardProtocols []string, idleTimeout, streamCreationTimeout time.Duration) error {

	_, err := httpstream.Handshake(req, w, supportedPortForwardProtocols)

	// negotiated protocol isn't currently used server side, but could be in the future

	if err != nil {

		// Handshake writes the error to the client

		return err

	}

	streamChan := make(chan httpstream.Stream, 1)

    klog.V(5).Infof("Upgrading port forward response")
    upgrader := spdy.NewResponseUpgrader()
    conn := upgrader.UpgradeResponse(w, req, httpStreamReceived(streamChan))
    if conn == nil {
        return errors.New("Unable to upgrade httpstream connection")
    }
    defer conn.Close()

    klog.V(5).Infof("(conn=%p) setting port forwarding streaming connection idle timeout to %v", conn, idleTimeout)
    conn.SetIdleTimeout(idleTimeout)

    h := &httpStreamHandler{
        conn:                  conn,
        streamChan:            streamChan,
        streamPairs:           make(map[string]*httpStreamPair),
        streamCreationTimeout: streamCreationTimeout,
        pod:                   podName,
        uid:                   uid,
        forwarder:             portForwarder,
    }
    h.run()

    return nil
    }



// UpgradeResponse upgrades an HTTP response to one that supports multiplexed
// streams. newStreamHandler will be called synchronously whenever the
// other end of the upgraded connection creates a new stream.
func (u responseUpgrader) UpgradeResponse(w http.ResponseWriter, req *http.Request, newStreamHandler httpstream.NewStreamHandler) httpstream.Connection {
	connectionHeader := strings.ToLower(req.Header.Get(httpstream.HeaderConnection))
	upgradeHeader := strings.ToLower(req.Header.Get(httpstream.HeaderUpgrade))
	if !strings.Contains(connectionHeader, strings.ToLower(httpstream.HeaderUpgrade)) || !strings.Contains(upgradeHeader, strings.ToLower(HeaderSpdy31)) {
		errorMsg := fmt.Sprintf("unable to upgrade: missing upgrade headers in request: %#v", req.Header)
		http.Error(w, errorMsg, http.StatusBadRequest)
		return nil
	}

	hijacker, ok := w.(http.Hijacker)
	if !ok {
		errorMsg := fmt.Sprintf("unable to upgrade: unable to hijack response")
		http.Error(w, errorMsg, http.StatusInternalServerError)
		return nil
	}

	w.Header().Add(httpstream.HeaderConnection, httpstream.HeaderUpgrade)
	w.Header().Add(httpstream.HeaderUpgrade, HeaderSpdy31)
	w.WriteHeader(http.StatusSwitchingProtocols)

	conn, bufrw, err := hijacker.Hijack()
	if err != nil {
		runtime.HandleError(fmt.Errorf("unable to upgrade: error hijacking response: %v", err))
		return nil
	}

	connWithBuf := &connWrapper{Conn: conn, bufReader: bufrw.Reader}
	spdyConn, err := NewServerConnection(connWithBuf, newStreamHandler)
	if err != nil {
		runtime.HandleError(fmt.Errorf("unable to upgrade: error creating SPDY server connection: %v", err))
		return nil
	}

	return spdyConn
}




```


可以发现，kubelet也使用了spdy协议（spdyConn）来复用http连接，回应kubectl 的spdy stream。

```c
	// run is the main loop for the httpStreamHandler. It processes new

// streams, invoking portForward for each complete stream pair. The loop exits

// when the httpstream.Connection is closed.

func (h *httpStreamHandler) run() {

	klog.V(5).Infof("(conn=%p) waiting for port forward streams", h.conn)

Loop:

	for {

		select {

		case <-h.conn.CloseChan():

			klog.V(5).Infof("(conn=%p) upgraded connection closed", h.conn)

			break Loop

		case stream := <-h.streamChan:

			requestID := h.requestID(stream)

			streamType := stream.Headers().Get(api.StreamType)

			klog.V(5).Infof("(conn=%p, request=%s) received new stream of type %s", h.conn, requestID, streamType)

            p, created := h.getStreamPair(requestID)
            if created {
                go h.monitorStreamPair(p, time.After(h.streamCreationTimeout))
            }
            if complete, err := p.add(stream); err != nil {
                msg := fmt.Sprintf("error processing stream for request %s: %v", requestID, err)
                utilruntime.HandleError(errors.New(msg))
                p.printError(msg)
            } else if complete {
                //真正转发的流程
                go h.portForward(p)
            }
        }
    }
    }
```







```c


// portForward invokes the httpStreamHandler's forwarder.PortForward
// function for the given stream pair.
func (h *httpStreamHandler) portForward(p *httpStreamPair) {
	defer p.dataStream.Close()
	defer p.errorStream.Close()

	portString := p.dataStream.Headers().Get(api.PortHeader)
	port, _ := strconv.ParseInt(portString, 10, 32)

	klog.V(5).Infof("(conn=%p, request=%s) invoking forwarder.PortForward for port %s", h.conn, p.requestID, portString)
	err := h.forwarder.PortForward(h.pod, h.uid, int32(port), p.dataStream)
	klog.V(5).Infof("(conn=%p, request=%s) done invoking forwarder.PortForward for port %s", h.conn, p.requestID, portString)

	if err != nil {
		msg := fmt.Errorf("error forwarding port %d to pod %s, uid %v: %v", port, h.pod, h.uid, err)
		utilruntime.HandleError(msg)
		fmt.Fprint(p.errorStream, msg.Error())
	}
}

//最终来到这里，即上面看到的streamingRuntime的PortForward函数：
	err := h.forwarder.PortForward(h.pod, h.uid, int32(port), p.dataStream)



	
func (a *criAdapter) PortForward(podName string, podUID types.UID, port int32, stream io.ReadWriteCloser) error {
	return a.Runtime.PortForward(podName, port, stream)
}


	//使用的runtime是
        streamingRuntime: &streamingRuntime{
			client:      client,
			execHandler: &NativeExecHandler{},
		},

func (r *streamingRuntime) PortForward(podSandboxID string, port int32, stream io.ReadWriteCloser) error {
	if port < 0 || port > math.MaxUint16 {
		return fmt.Errorf("invalid port %d", port)
	}
	return portForward(r.client, podSandboxID, port, stream)
}


	
```
最后分析下portForward(r.client, podSandboxID, port, stream)函数的流程：

(1)首先使用dockerclient  inspect容器的状态

(2)获取container的Pid

(3)使用socat - TCP4:localhost:port连接远程端口

(4)使用io.copy转发从api-server转发过来的消息

```c
func portForward(client libdocker.Interface, podSandboxID string, port int32, stream io.ReadWriteCloser) error {
	container, err := client.InspectContainer(podSandboxID)
	if err != nil {
		return err
	}

	if !container.State.Running {
		return fmt.Errorf("container not running (%s)", container.ID)
	}

	containerPid := container.State.Pid
	socatPath, lookupErr := exec.LookPath("socat")
	if lookupErr != nil {
		return fmt.Errorf("unable to do port forwarding: socat not found.")
	}

	args := []string{"-t", fmt.Sprintf("%d", containerPid), "-n", socatPath, "-", fmt.Sprintf("TCP4:localhost:%d", port)}

	nsenterPath, lookupErr := exec.LookPath("nsenter")
	if lookupErr != nil {
		return fmt.Errorf("unable to do port forwarding: nsenter not found.")
	}

	commandString := fmt.Sprintf("%s %s", nsenterPath, strings.Join(args, " "))
	klog.V(4).Infof("executing port forwarding command: %s", commandString)

	command := exec.Command(nsenterPath, args...)
	command.Stdout = stream

	stderr := new(bytes.Buffer)
	command.Stderr = stderr

	// If we use Stdin, command.Run() won't return until the goroutine that's copying
	// from stream finishes. Unfortunately, if you have a client like telnet connected
	// via port forwarding, as long as the user's telnet client is connected to the user's
	// local listener that port forwarding sets up, the telnet session never exits. This
	// means that even if socat has finished running, command.Run() won't ever return
	// (because the client still has the connection and stream open).
	//
	// The work around is to use StdinPipe(), as Wait() (called by Run()) closes the pipe
	// when the command (socat) exits.
	inPipe, err := command.StdinPipe()
	if err != nil {
		return fmt.Errorf("unable to do port forwarding: error creating stdin pipe: %v", err)
	}
	go func() {
		io.Copy(inPipe, stream)
		inPipe.Close()
	}()

	if err := command.Run(); err != nil {
		return fmt.Errorf("%v: %s", err, stderr.String())
	}

	return nil
}

```


## 总结：

由上面的分析可以发现，portford命令经历了重重转发，最终才能到达容器里面。

![](https://raw.githubusercontent.com/liucimin/liucimin.github.io/master/_posts/asserts/images/k8sportfowardstruct.png)

梳理下来就是：

kubectl 创建本地连接等待用户连接，创建和api-server的连接用于转发用户请求，使用spdy协议发送请求

kube-apiserver 接收kubectl的post portford请求，查询pod对应的kubelet和node的位置，连接pod对应的node下的kubelet的dockerserver，创建proxy直接转发kubectl中发送的用户的http消息

kubelet  使用dockerserver创建server等待api-server的连接，与连接使用spdy协议交互，直接进入容器命名空间使用socat与容器对应端口通信，将收到的消息全部转发到容器对应port。













​	