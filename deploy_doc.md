kubectl create secret generic my-app-db-creds \
  --from-literal=username=myuser \
  --from-literal=password=supersecretpassword

kubectl apply -f postgres-cluster.yaml


Step 3: How to Connect (The "CNPG Way")
CNPG automatically creates two services for your application to use. You should never hardcode IP addresses or pod names.
For Read/Write traffic (Primary):
Use the service named <cluster-name>-rw.[2]
Host: my-app-db-rw
Example: postgres://myuser:password@my-app-db-rw:5432/myapp
For Read-Only traffic (Replicas):
Use the service named <cluster-name>-ro.[2] CNPG automatically load-balances your SELECT queries across both of your read-only replicas.
Host: my-app-db-ro
Example: postgres://myuser:password@my-app-db-ro:5432/myapp
