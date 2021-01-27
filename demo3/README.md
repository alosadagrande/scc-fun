# Linux Capabilities

In this demo we will learn how we can work with capabilities inside our containers.

We will start by restricting the set of capabilities available in a local container with podman, and then we will see how we can manage capabilities for pods running on OpenShift.

## Managing Capabilities with Podman

We will see how we can add/drop capabilities in a given container using Podman and the implications of doing so.

### Hands-on Demo

1. Install podman on your system in case you don't have it yet.
2. Let's run an nginx container and see which capabilities are added to the container.

    ~~~sh
    podman run -d --rm --name nginx-cap-test nginx:latest
    ~~~
3. Let's check the capabilities assigned to that process.

    1. We can do it using the `/proc` filesystem:

        ~~~sh
        CONTAINER_PID=$(podman inspect nginx-cap-test --format {{.State.Pid}})
        cat /proc/${CONTAINER_PID}/status | grep Cap
        ~~~

        ~~~
        CapInh:	00000000a80425fb
        CapPrm:	00000000a80425fb
        CapEff:	00000000a80425fb
        CapBnd:	00000000a80425fb
        CapAmb:	00000000a80425fb
        ~~~
    2. We got a list of capabilities, if we want to decode the value we can use `capsh`:

        ~~~sh
        capsh --decode=00000000a80425fb
        ~~~

        ~~~
        0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
        ~~~
    3. We can get them with `podman inspect` as well:

        ~~~sh
        podman inspect nginx-cap-test --format {{.EffectiveCaps}}
        ~~~

        ~~~
        [CAP_AUDIT_WRITE CAP_CHOWN CAP_DAC_OVERRIDE CAP_FOWNER CAP_FSETID CAP_KILL CAP_MKNOD CAP_NET_BIND_SERVICE CAP_NET_RAW CAP_SETFCAP CAP_SETGID CAP_SETPCAP CAP_SETUID CAP_SYS_CHROOT]
        ~~~
    4. And a third tool to get them could be `getpcaps`:

        ~~~sh
        CONTAINER_PID=$(podman inspect nginx-cap-test --format {{.State.Pid}})
        getpcaps ${CONTAINER_PID}
        ~~~
    5. We can stop the container now:

        ~~~sh
        podman stop nginx-cap-test
        ~~~
4. So by default, podman will configure the capabilities defined [here](https://github.com/containers/common/blob/v0.33.1/pkg/config/default.go#L62-L77) or alternatively the ones configured in its configuration file.

We can also drop capabilities not needed by our application, that way we are reducing the attack surface. Let's see.

1. We know that our demo application doesn't require any capability to run by default, so we can drop all capabilities:

    ~~~sh
    podman run -d --rm --name reverse-words-app --cap-drop=all quay.io/mavazque/reversewords:latest
    ~~~
2. If we check the capabilities, we will see that the container has no capabilities:

    ~~~sh
    podman inspect reverse-words-app --format {{.EffectiveCaps}}
    ~~~

    ~~~
    []
    ~~~
3. Now we want our application to bind port 80, that will require the capability `NET_BIND_SERVICE`, let's see what happens if we try to run the app without that capability:

    ~~~sh
    podman run --rm --name reverse-words-app-80 --cap-drop=all -e APP_PORT=80 quay.io/mavazque/reversewords:latest
    ~~~

    1. The container failed to start because it couldn't bind port 80:
      
        ~~~
        2021/01/19 16:12:53 Starting Reverse Api v0.0.17 Release: NotSet
        2021/01/19 16:12:53 Listening on port 80
        2021/01/19 16:12:53 listen tcp :80: bind: permission denied
        ~~~
4. If we add that capability, we will see how the container now starts properly and binds to port 80:

    ~~~sh
    podman run --rm --name reverse-words-app-80 --cap-drop=all --cap-add=cap_net_bind_service -e APP_PORT=80 quay.io/mavazque/reversewords:latest
    ~~~

    ~~~
    2021/01/19 16:14:34 Starting Reverse Api v0.0.17 Release: NotSet
    2021/01/19 16:14:34 Listening on port 80
    ~~~

## Managing Capabilities on OpenShift

Now it's time to see how capabilities can be managed on OpenShift.

### Hands-on Demo 1

In this demo we are going to see how we can drop all capabilities but NET_BIND_SERVICE on the pod running our application.

1. Create a new namespace for running our tests:

    ~~~sh
    NAMESPACE=test-capabilities
    oc create ns ${NAMESPACE}
    ~~~
2. Create a user and give it edit role on the namespace

    ~~~sh
    oc -n ${NAMESPACE} create sa testuser
    oc -n ${NAMESPACE} adm policy add-role-to-user edit system:serviceaccount:${NAMESPACE}:testuser
    ~~~
3. The default SCC `restricted` has the following settings for capabilities:

    > **NOTE**: The configuration below basically means pods running with restricted SCC cannot gain any new capabilities. And KILL, MKNOD, SETUID and SETGID are dropped in case those are granted by the container runtime.

    ~~~yaml
    defaultAddCapabilities: null
    allowedCapabilities: null
    requiredDropCapabilities:
    - KILL
    - MKNOD
    - SETUID
    - SETGID
    ~~~

4. We are going to create our own SCC based on the restricted one, and on top of that we need out container to run with with `anyuid` (due to the issue mentioned earlier) and we want to be able to use `NET_BIND_SERVICE` capability.

    > **NOTE**: We removed `system:authenticated` group so we will need to assign the SCC manually to our SAs/Users/Groups.
    ~~~sh
    cat <<EOF | oc create -f -
    kind: SecurityContextConstraints
    metadata:
      name: restricted-netbind
    priority: null
    readOnlyRootFilesystem: false
    requiredDropCapabilities:
    - KILL
    - MKNOD
    - SETUID
    - SETGID
    runAsUser:
      type: RunAsAny
    seLinuxContext:
      type: MustRunAs
    supplementalGroups:
      type: RunAsAny
    users: []
    volumes:
    - configMap
    - downwardAPI
    - emptyDir
    - persistentVolumeClaim
    - projected
    - secret
    allowHostDirVolumePlugin: false
    allowHostIPC: false
    allowHostNetwork: false
    allowHostPID: false
    allowHostPorts: false
    allowPrivilegeEscalation: true
    allowPrivilegedContainer: false
    allowedCapabilities:
    - NET_BIND_SERVICE
    apiVersion: security.openshift.io/v1
    defaultAddCapabilities: null
    fsGroup:
      type: MustRunAs
    groups: []
    EOF
    ~~~
5. We're grating use privileges of this new SCC to the SA test-user

    ~~~sh
    oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-netbind system:serviceaccount:${NAMESPACE}:testuser
    ~~~
5. On top of the SCC caps we can drop/add (add will depend on the SCC settings) capabilities on a given pod:

    > **NOTE**: Below example drops NET_BIND_SERVICE capability
    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: reversewords-app-captest
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/mavazque/reversewords:latest
        name: reversewords
        securityContext:
          capabilities:
            drop:
            - NET_BIND_SERVICE
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
4. Since we're not binding to a privileged port, the application will start with no issues. On top of that the pod started with the `restricted` SCC since it didn't need any extra config provided by our new SCC. Now, let's see what happens if we create the pod with a binding to port 80:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: reversewords-app-captest-80
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/mavazque/reversewords:latest
        name: reversewords
        env:
        - name: APP_PORT
          value: "80"
        securityContext:
          runAsUser: 0
          capabilities:
            drop:
            - NET_BIND_SERVICE
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~

    1. The pod failed to run, in the logs we can see:

      ~~~
      2021/01/19 17:12:39 Starting Reverse Api v0.0.17 Release: NotSet
      2021/01/19 17:12:39 Listening on port 80
      2021/01/19 17:12:39 listen tcp :80: bind: permission denied
      ~~~
5. If we drop all capabilities and we add NET_BIND_SERVICE to the list of capabilities we will see how the pod now runs properly:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: reversewords-app-captest-80-2
    spec:
      serviceAccountName: testuser
      containers:
      - image: quay.io/mavazque/reversewords:latest
        name: reversewords
        env:
        - name: APP_PORT
          value: "80"
        securityContext:
          capabilities:
            drop:
            - all
            add:
            - NET_BIND_SERVICE
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~

### Hands-on Demo 2

In this demo we need an SCC so we can run a pod that changes the ownership of the `/etc/resolv.conf` file to `nobody` user. The information we have is that `CHOWN` capability will be required for `chown` to work.

1. Let's try to run the pod with the SCC we have already in place:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: chown-test
    spec:
      serviceAccountName: testuser
      containers:
      - image: registry.centos.org/centos:8
        command: ["chown", "-v", "nobody", "/etc/resolv.conf"]
        name: centos
        securityContext:
          capabilities:
            drop:
            - all
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
2. The pod failed:

    ~~~
    chown: changing ownership of '/etc/resolv.conf': Operation not permitted
    failed to change ownership of '/etc/resolv.conf' from root to nobody
    ~~~
3. We can patch the SCC we created in the previous demo and allow the use of CHOWN capability as well:

    ~~~sh
    oc patch scc restricted-netbind -p '{"allowedCapabilities":["NET_BIND_SERVICE","CHOWN"]}' --type=merge
    ~~~
4. Let's try to create the pod again, but now request the capability we just added to the SCC:

    ~~~sh
    cat <<EOF | oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: chown-test-2
    spec:
      serviceAccountName: testuser
      containers:
      - image: registry.centos.org/centos:8
        command: ["chown", "-v", "nobody", "/etc/resolv.conf"]
        name: centos
        securityContext:
          capabilities:
            drop:
            - all
            add:
            - CHOWN
      dnsPolicy: ClusterFirst
      restartPolicy: Never
    status: {}
    EOF
    ~~~
5. Now we can see that it works as expected now:

    ~~~
    changed ownership of '/etc/resolv.conf' from root to nobody
    ~~~