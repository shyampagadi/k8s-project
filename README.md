Here’s a detailed capstone project that incorporates various Kubernetes topics you’ve covered, including Deployments, Services, Persistent Volumes, StatefulSets, Ingress, ConfigMaps, and Secrets. This project aims to create a **Multi-tier Voting Application** that simulates a real-world production environment.

## Project Overview: Multi-tier Voting Application

### Project Components
1. **Frontend**: A web application where users can cast their votes.
2. **Backend API**: A RESTful API to handle voting logic and store results.
3. **Database**: A PostgreSQL database to persist votes and results.
4. **Ingress**: To route traffic to the frontend and backend services.
5. **ConfigMaps and Secrets**: To manage application configurations and sensitive information.

### Architecture Diagram

```
┌──────────────────┐       ┌──────────────────┐
│                  │       │                  │
│   Users/Clients  │       │    Ingress       │
│                  │       │                  │
└──────────────────┘       └──────────────────┘
            │                            │
            │                            │
            │                            │
            │            ┌──────────────┼──────────────┐
            │            │                             │
            ▼            ▼                             ▼
 ┌────────────────┐  ┌────────────────┐      ┌────────────────┐
 │                │  │                │      │                │
 │  Frontend App  │  │ Backend API    │      │   PostgreSQL   │
 │                │  │                │      │   Database      │
 └────────────────┘  └────────────────┘      └────────────────┘
```

### Prerequisites
- Kubernetes cluster (local or cloud-based)
- kubectl installed and configured
- Docker installed
- Knowledge of Helm (optional)

### Step 1: Setting Up the Project Directory
Create a project directory with the following structure:
```
voting-app/
├── frontend/
│   ├── Dockerfile
│   └── src/
├── backend/
│   ├── Dockerfile
│   └── src/
├── database/
│   └── init.sql
├── k8s/
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   ├── postgres-deployment.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── secret.yaml
└── README.md
```

### Step 2: Frontend Application
#### 2.1 Create Frontend Application

Use a simple React app for the frontend.

**Dockerfile for Frontend**:
```dockerfile
# Frontend Dockerfile
FROM node:14 AS build
WORKDIR /app
COPY ./src /app/src
COPY package.json /app/
RUN npm install
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 2.2 Frontend Source Code

You can use this simple React example for your `src` folder. Create an `index.js` file:
```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';

const App = () => {
  return (
    <div>
      <h1>Voting Application</h1>
      <p>Vote for your favorite option!</p>
      {/* Voting Logic Goes Here */}
    </div>
  );
};

ReactDOM.render(<App />, document.getElementById('root'));
```

### Step 3: Backend Application
#### 3.1 Create Backend Application

Use a simple Node.js Express API for the backend.

**Dockerfile for Backend**:
```dockerfile
# Backend Dockerfile
FROM node:14
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

#### 3.2 Backend Source Code

In your backend folder, create a simple Express server. Here's a basic `server.js` file:
```javascript
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(bodyParser.json());

let votes = {};

app.post('/vote', (req, res) => {
  const { option } = req.body;
  votes[option] = (votes[option] || 0) + 1;
  res.json({ success: true, votes });
});

app.get('/results', (req, res) => {
  res.json(votes);
});

app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
```

### Step 4: Database Initialization
#### 4.1 Create Database Initialization Script

In the `database` folder, create an `init.sql` file to set up your PostgreSQL database:
```sql
CREATE TABLE votes (
    option VARCHAR(50) NOT NULL,
    count INT DEFAULT 0
);
```

### Step 5: Kubernetes Deployment Files
#### 5.1 ConfigMap and Secret
Create `k8s/configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  BACKEND_URL: "http://backend-service:3000"
```

Create `k8s/secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  POSTGRES_USER: <base64-encoded-username>
  POSTGRES_PASSWORD: <base64-encoded-password>
```

#### 5.2 Deployments and Services
Create `k8s/frontend-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: your-frontend-image:latest
          ports:
            - containerPort: 80
```

Create `k8s/backend-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: your-backend-image:latest
          env:
            - name: BACKEND_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: BACKEND_URL
          ports:
            - containerPort: 3000
```

Create `k8s/postgres-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:latest
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: POSTGRES_PASSWORD
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

### Step 6: Persistent Volume and Claim
Create a Persistent Volume and Claim in `k8s/postgres-pv-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/postgres
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Step 7: Ingress Configuration
Create `k8s/ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: voting-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app1.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: app2.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 3000
```

### Step 8: Deploying the Application
1. **Apply the Persistent Volume and Claim**:
   ```bash
   kubectl apply -f k8s/postgres-pv-pvc.yaml
   ```

2. **Create the ConfigMap and Secret**:
   ```bash


   kubectl apply -f k8s/configmap.yaml
   kubectl apply -f k8s/secret.yaml
   ```

3. **Deploy the Frontend and Backend**:
   ```bash
   kubectl apply -f k8s/frontend-deployment.yaml
   kubectl apply -f k8s/backend-deployment.yaml
   kubectl apply -f k8s/postgres-deployment.yaml
   ```

4. **Apply the Ingress Configuration**:
   ```bash
   kubectl apply -f k8s/ingress.yaml
   ```

### Step 9: Accessing the Application
1. Update your `/etc/hosts` file (or DNS) to point the domain names (e.g., `app1.example.com`, `app2.example.com`) to the IP of your Ingress controller.
2. Open your web browser and access the application using the configured domain names.

### Step 10: Monitoring and Logging (Optional)
Integrate a monitoring tool like Prometheus and Grafana to visualize metrics and logs from your application. You can set up sidecar containers or use a logging operator to collect logs.

### Conclusion
This capstone project covers multiple Kubernetes topics and simulates a real-world application deployment. You can further extend this project by adding features like authentication, scaling, automated CI/CD pipelines, or integrating with a cloud provider for production-grade features. 

Feel free to ask if you need any specific resources, links, or additional details on any of the steps!
