# Auth Server Kubernetes Configuration (हिंदी गाइड)

## 1. ConfigMap (`configmap.yaml`)

यह फाइल एप्लिकेशन की **सेटिंग्स** (धाल) को स्टोर करती है जो गुप्त (secret) नहीं हैं।

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-server-config
  namespace: backend
data:
  SERVICE_NAME: auth-server
  PORT: "3000"
  LOG_LEVEL: info
```


- **`apiVersion: v1`**: यह कुबेरनेट्स को बताता है कि हम API का `v1` संस्करण उपयोग कर रहे हैं।
- **`kind: ConfigMap`**: यह फाइल का प्रकार बताता है। `ConfigMap` का मतलब है "कॉन्फ़िगरेशन का नक्शा"।
- **`metadata`**: इस फाइल के बारे में जानकारी।
  - **`name: auth-server-config`**: इस कॉन्फ़िग का नाम। हम इसी नाम से इसे डिप्लॉयमेंट (Deployment) में बुलाएंगे।
  - **`namespace: backend`**: यह कॉन्फ़िग `backend` नाम के समूह/फोल्डर में बनेगा।
- **`data`**: यहाँ हम अपने वेरिएबल्स (variables) लिखते हैं।
  - **`SERVICE_NAME` और `PORT`**: ये वो वेरिएबल्स हैं जो आपके कोड (`process.env`) में मिलेंगे। `PORT: "3000"` बताता है कि ऐप 3000 पोर्ट पर चलेगा।

---

## 2. Secret (`secret.yaml`)

यह फाइल **पासवर्ड और कीज़** (passwords & keys) को स्टोर करती है जिन्हें हम सबको नहीं दिखाना चाहते।

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-server-secret
  namespace: backend
type: Opaque
stringData:
  JWT_SECRET: my-super-secret
  DATABASE_URL: mongodb://user:pass@db:27017/auth
```


- **`kind: Secret`**: यह बताता है कि यह फाइल एक `Secret` है, इसका डेटा सुरक्षित रहेगा।
- **`type: Opaque`**: यह साधारण की-वैल्यू (key-value) डेटा स्टोर करने का मानक तरीका है।
- **`stringData`**: इसका मतलब है हम डेटा को साधारण टेक्स्ट (plain text) में लिख रहे हैं। कुबेरनेट्स इसे खुद एन्क्रिप्ट/एनकोड (encrypt/encode) कर लेगा ताकि यह सीधे पढ़ा न जा सके।
  - **`JWT_SECRET`**: लॉगिन टोकन बनाने के लिए गुप्त कुंजी (secret key)।
  - **`DATABASE_URL`**: डेटाबेस का पासवर्ड और URL।

---

## 3. Deployment (`deployment.yaml`)

यह सबसे मुख्य फाइल है। यह बताती है कि **ऐप कैसे रन होगा**, कितनी कॉपी चलेंगी, और क्रैश (crash) होने पर क्या होगा।

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
  namespace: backend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: auth-server
  template:
    metadata:
      labels:
        app: auth-server
    spec:
      containers:
        - name: auth-server
          image: your-dockerhub/auth-server:1.0.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: auth-server-config
            - secretRef:
                name: auth-server-secret
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
```

### लाइन-दर-लाइन व्याख्या (हिंदी में)

- **`kind: Deployment`**: यह बताता है कि हम एप्लिकेशन डिप्लॉय कर रहे हैं। यह पॉड्स (Pods) को मैनेज करता है।
- **`spec`**: डिप्लॉयमेंट के नियम (rules)।
  - **`replicas: 2`**: इसका मतलब है हमेशा **2 कॉपी (pods)** चलेंगी। अगर एक फेल हो गई, तो दूसरी चलती रहेगी।
  - **`strategy: RollingUpdate`**: जब हम नया वर्जन डिप्लॉय करेंगे, तो पुराना बंद करने से पहले नया स्टार्ट होगा ताकि उपयोगकर्ताओं को रुकावट न मिले।
- **`selector`**: यह बताता है कि डिप्लॉयमेंट किन पॉड्स का ध्यान रखेगा (जिन पर `app: auth-server` लेबल लगा हो)।
- **`template`**: यह ब्लूप्रिंट (blueprint) है कि नया पॉड कैसा दिखेगा।
  - **`containers`**: पॉड के अंदर चलने वाले कंटेनर की सेटिंग्स।
    - **`image`**: आपके ऐप का डॉकर इमेज (Docker Image) कहाँ से उठाना है।
    - **`ports`**: कंटेनर का पोर्ट 3000 खुला रहेगा।
    - **`envFrom`**: एनवायरनमेंट वेरिएबल्स कहाँ से लेने हैं?
      - **`configMapRef`**: `auth-server-config` फाइल से सेटिंग्स ले लो।
      - **`secretRef`**: `auth-server-secret` फाइल से पासवर्ड ले लो।
    - **`resources`**: ऐप कितना CPU/RAM लेगा?
      - **`requests`**: शुरू होने के लिए कम से कम इतना CPU/RAM चाहिए।
      - **`limits`**: इससे ज़्यादा इस्तेमाल किया तो कुबेरनेट्स ऐप को रीस्टार्ट (restart) कर देगा।
    - **`readinessProbe`**: कुबेरनेट्स चेक करेगा (`/health` पर) कि क्या ऐप रिक्वेस्ट लेने के लिए तैयार है?
    - **`livenessProbe`**: कुबेरनेट्स चेक करेगा कि क्या ऐप ज़िंदा है? अगर हैंग (hang) हो गया, तो रीस्टार्ट कर देगा।

---

## 4. Service (`service.yaml`)

यह फाइल **नेटवर्किंग** (Networking) के लिए है। पॉड्स के IP बदलते रहते हैं, लेकिन सर्विस एक फिक्स एड्रेस (fixed address) देती है।

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-server
  namespace: backend
spec:
  type: ClusterIP
  selector:
    app: auth-server
  ports:
    - port: 80
      targetPort: 3000
```

### लाइन-दर-लाइन व्याख्या (हिंदी में)

- **`kind: Service`**: यह एक नेटवर्क सर्विस है।
- **`spec`**: सर्विस के नियम।
  - **`type: ClusterIP`**: इसका मतलब है यह सर्विस सिर्फ **क्लस्टर के अंदर** ही एक्सेस हो सकती है (बाहर से नहीं)।
  - **`selector`**: यह सर्विस अपना ट्रैफिक उन पॉड्स पर भेजेगी जिनका लेबल `app: auth-server` है।
  - **`ports`**:
    - **`port: 80`**: सर्विस का दरवाज़ा। दूसरे एप्स `http://auth-server:80` पर रिक्वेस्ट भेजेंगे।
    - **`targetPort: 3000`**: सर्विस वो ट्रैफिक उठाकर अंदर कंटेनर के पोर्ट 3000 पर भेज देगी।

---

### कैसे रन करें (How to Apply)

इन सभी फाइलों को कुबेरनेट्स क्लस्टर पर लागू करने के लिए यह कमांड चलाएं:

```bash
kubectl apply -f .
```

इससे वो चारों चीज़ें (ConfigMap, Secret, Deployment, Service) बन जाएंगी।
