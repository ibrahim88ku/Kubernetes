# k8s-cluster
This is k8s cluster
- This is my test project on k8s
    - testhhhh
        - testjjjj
    ```
    apiVersion: v1
    kind: Pod
    metadata:
     name: nginx
     labels:
      name: nginx
    spec:
     containers:
     - name: nginx
       image: nginx
       ports:
       - containerPort: 8080
     nodeName: node02
    ```
    