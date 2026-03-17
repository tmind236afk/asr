# Deploy Riva NIM Parakeet ASR on GKE

Step-by-step commands to deploy NVIDIA Riva NIM with Parakeet 1.1B CTC on GKE,
designed to be run directly in [Google Cloud Shell](https://shell.cloud.google.com). Works on a remote session as well.

---

## Step 0 — Set Variables

Copy this entire block, replace the placeholder values, and paste into Cloud Shell:

```bash
# ── EDIT THESE ──
export PROJECT_ID=""
export ZONE=""
export NGC_API_KEY=""

# ── Leave these as-is ──
export CLUSTER_NAME="riva-asr-cluster"
export RIVA_MODEL="parakeet-1-1b-ctc-en-us"

gcloud config set project $PROJECT_ID
gcloud config set compute/zone $ZONE
```

If you haven't created the cluster yet, run this first:

```bash
gcloud container clusters create $CLUSTER_NAME \
    --zone $ZONE \
    --machine-type g2-standard-8 \
    --accelerator type=nvidia-l4,count=1 \
    --num-nodes 1 \
    --scopes cloud-platform \
    --quiet
```

This takes 3-5 minutes. Then continue from Step 1 below.
---

## Step 1 — Connect to the Cluster

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
kubectl get nodes
```

You should see your node(s) listed. Verify the GPU is available:

```bash
kubectl get nodes -o=custom-columns='NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Expected output should show `1` under the GPU column.

---

## Step 2 — Verify NVIDIA GPU Drivers

In recent GKE versions (1.27.2+), Google automatically installs the NVIDIA drivers and device plugin for you. 

You can verify the GPU is registered and ready by checking if `nvidia.com/gpu` is allocatable:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\tGPUs: "}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
```

The output should show `GPUs: 1`. Once you confirm this, you can proceed!

---

## Step 3 — Create NGC Secrets

These secrets allow the cluster to pull the Riva container from NGC and download model weights:

```bash
# Image pull secret
kubectl create secret docker-registry ngc-secret \
    --docker-server=nvcr.io \
    --docker-username='$oauthtoken' \
    --docker-password=$NGC_API_KEY \
    --dry-run=client -o yaml | kubectl apply -f -

# API key secret
kubectl create secret generic ngc-api \
    --from-literal=NGC_API_KEY=$NGC_API_KEY \
    --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secrets
```

You should see `ngc-secret` and `ngc-api` listed.

---

## Step 4 — Install Helm and Add NVIDIA Repo

Cloud Shell has Helm pre-installed. **If you are running this on a remote server** and `helm` is not found, install it first:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
sudo ./get_helm.sh
```

Add the NVIDIA NIM chart repository:

```bash
helm repo add nim https://helm.ngc.nvidia.com/nim/nvidia \
    --username='$oauthtoken' --password=$NGC_API_KEY

helm repo update nim
```

Verify the Riva chart is available:

```bash
helm search repo nim/riva-nim
```

---

## Step 5 — Create the Helm Values File

```bash
cat > /tmp/riva-values.yaml << 'EOF'
image:
  repository: nvcr.io/nim/nvidia/parakeet-1-1b-ctc-en-us
  tag: latest
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ngc-secret

nim:
  ngcAPISecret: ngc-api

env:
  - name: NIM_TAGS_SELECTOR
    value: "name=parakeet-1-1b-ctc-en-us,mode=str,vad=default"

resources:
  limits:
    nvidia.com/gpu: 1
  requests:
    nvidia.com/gpu: 1

service:
  type: LoadBalancer
  ports:
    - name: http
      port: 9000
      targetPort: 9000
      protocol: TCP
    - name: grpc
      port: 50051
      targetPort: 50051
      protocol: TCP

persistence:
  enabled: true
  size: 50Gi

startupProbe:
  httpGet:
    path: /v1/health/ready
    port: 9000
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 120
  timeoutSeconds: 10

livenessProbe:
  httpGet:
    path: /v1/health/ready
    port: 9000
  periodSeconds: 30
  failureThreshold: 3
  timeoutSeconds: 10
EOF
```

---

## Step 6 — Deploy Riva NIM

```bash
helm install riva-asr nim/riva-nim -f /tmp/riva-values.yaml
```

This starts the deployment. The pod will download the model and optimize it
with TensorRT on first run. This can take a while as model optimizations take place (**20-40 minutes**).

---

## Step 7 — Monitor Deployment

Watch the pod status until it shows `Running` and `1/1 READY`:

```bash
kubectl get pods -w
```

(Press `Ctrl+C` to stop watching once the pod is ready.)

To see detailed progress (model download, TensorRT optimization):

```bash
kubectl logs -f -l app.kubernetes.io/instance=riva-asr
```

If the pod is stuck in `Pending`, check events:

```bash
kubectl describe pod -l app.kubernetes.io/instance=riva-asr
```

---

## Step 8 — Get the External IP

Once the pod is running, get the LoadBalancer external IP:

```bash
kubectl get svc riva-asr-riva-nim
```

Look for the `EXTERNAL-IP` column. If it shows `<pending>`, wait a minute and re-run. Save it to a variable:

```bash
export EXTERNAL_IP=$(kubectl get svc riva-asr-riva-nim \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "External IP: $EXTERNAL_IP"
```

> **Note:** If your `EXTERNAL-IP` stays stuck in `<pending>` indefinitely, your GCP organization may have a strict security policy (`compute.restrictLoadBalancerCreationForTypes`) blocking external load balancers. 
> 
> **Workaround:** You can skip the LoadBalancer entirely and securely forward the traffic directly to your local machine. Open a second terminal window and run:
> ```bash
> kubectl port-forward svc/riva-asr-riva-nim 9000:9000 50051:50051
> ```
> Leave that terminal open. Then, back in your main terminal, set your `EXTERNAL_IP` variable to localhost so the rest of the guide works:
> ```bash
> export EXTERNAL_IP="127.0.0.1"
> ```

---

## Step 9 — Verify Health

```bash
curl -s http://$EXTERNAL_IP:9000/v1/health/ready | python3 -m json.tool
```

Expected output:

```json
{
    "ready": true
}
```

---

## Step 10 — Generate Test WAV file

Generate a test WAV file and send it to the HTTP API:

```bash
# Install test dependencies (including system packages for audio conversion)
sudo apt-get update && sudo apt-get install -y ffmpeg
pip install -q gTTS pydub

# Generate test audio
python3 -c "
from gtts import gTTS
from pydub import AudioSegment
tts = gTTS('Hello, what is the weather like today?', lang='en')
tts.save('/tmp/test.mp3')
audio = AudioSegment.from_mp3('/tmp/test.mp3').set_frame_rate(16000).set_channels(1).set_sample_width(2)
audio.export('/tmp/test.wav', format='wav')
print('Test audio generated: /tmp/test.wav')
"

```

---

## Step 11 — Test gRPC Streaming

```bash
# Install Riva client
pip install -q nvidia-riva-client

# Clone the official Python client scripts
git clone --depth 1 https://github.com/nvidia-riva/python-clients.git /tmp/riva-clients 2>/dev/null

# Run streaming transcription
python3 /tmp/riva-clients/scripts/asr/transcribe_file.py \
    --server $EXTERNAL_IP:50051 \
    --language-code en-US \
    --input-file /tmp/test.wav
```

You should see interim and final transcription results printed as the audio
is streamed to the server.

For a more detailed streaming test with interim results visible:

```bash
python3 << 'PYEOF'
import os
import riva.client

EXTERNAL_IP = os.environ["EXTERNAL_IP"]

auth = riva.client.Auth(uri=f"{EXTERNAL_IP}:50051", use_ssl=False)
asr_service = riva.client.ASRService(auth)

config = riva.client.StreamingRecognitionConfig(
    config=riva.client.RecognitionConfig(
        encoding=riva.client.AudioEncoding.LINEAR_PCM,
        sample_rate_hertz=16000,
        audio_channel_count=1,
        language_code="en-US",
        max_alternatives=1,
        enable_automatic_punctuation=True,
        verbatim_transcripts=False,
    ),
    interim_results=True,
)

CHUNK_SIZE = 8000  # 250ms of 16kHz 16-bit mono

with open("/tmp/test.wav", "rb") as f:
    f.read(44)  # skip WAV header
    chunks = []
    while True:
        chunk = f.read(CHUNK_SIZE)
        if not chunk:
            break
        chunks.append(chunk)

print(f"Streaming {len(chunks)} chunks ...\n")

responses = asr_service.streaming_response_generator(
    audio_chunks=chunks,
    streaming_config=config,
)

final = ""
for resp in responses:
    for result in resp.results:
        text = result.alternatives[0].transcript
        if result.is_final:
            final += text
            print(f"  [FINAL]   {text}")
        else:
            print(f"  [INTERIM] {text}")

print(f"\nFull transcription: \"{final}\"")
PYEOF
```

---

## Step 12 — Test WebSocket Realtime API

```bash
pip install -q websocket-client

python3 << 'PYEOF'
import os, json, base64, threading, websocket, time

EXTERNAL_IP = os.environ["EXTERNAL_IP"]
ws_url = f"ws://{EXTERNAL_IP}:9000/v1/realtime?intent=transcription"
print(f"Connecting to {ws_url} ...\n")

results = []
done = threading.Event()

def on_message(ws, msg):
    event = json.loads(msg)
    t = event.get("type", "")
    if t == "conversation.item.input_audio_transcription.delta":
        text = event.get("delta", "")
        print(f"  [PARTIAL] {text}")
        results.append(text)
    elif t == "conversation.item.input_audio_transcription.completed":
        text = event.get("transcript", "")
        print(f"  [FINAL]   {text}")
        results.append(text)
        done.set()
    elif t == "error":
        print(f"  [ERROR]   {event}")
        done.set()

def stream_audio(ws):
    # Wait a tiny bit for the session update to register on the server
    time.sleep(0.5)
    with open("/tmp/test.wav", "rb") as f:
        f.read(44)  # Skip WAV header
        while True:
            chunk = f.read(8000) # 0.25 seconds of audio
            if not chunk:
                break
            ws.send(json.dumps({
                "type": "input_audio_buffer.append",
                "audio": base64.b64encode(chunk).decode(),
            }))
            # Sleep slightly to simulate real-time streaming
            time.sleep(0.1)
            
    ws.send(json.dumps({"type": "input_audio_buffer.commit"}))
    ws.send(json.dumps({"type": "input_audio_buffer.done"}))

def on_open(ws):
    ws.send(json.dumps({
        "type": "transcription_session.update",
        "session": {
            "input_audio_format": "pcm16",
            "input_audio_transcription": {
                "language": "en-US",
                "model": "parakeet-1.1b-en-US-asr-streaming"
            }
        }
    }))
    # Start streaming in a background thread so we don't block the WebSocket receive loop
    threading.Thread(target=stream_audio, args=(ws,), daemon=True).start()

def on_error(ws, err):
    print(f"  [ERROR] {err}")
    done.set()

app = websocket.WebSocketApp(ws_url, on_open=on_open,
                              on_message=on_message, on_error=on_error)
app.run_forever()

if results:
    print(f"\nFinal: \"{results[-1]}\"")
else:
    print("\nNo results (timeout)")
PYEOF
```

---

## Step 13 — Test End of Utterance (EoU) Detection

Riva handles "End of Utterance" (EoU) automatically using its built-in Voice Activity Detection (VAD) model. When a user stops speaking for a specific threshold of time, Riva immediately closes the current transcript, emits an `is_final=True` event, and prepares to start a brand new sentence.

To demonstrate this, we can generate a test audio file that has **speech**, followed by **2 seconds of pure silence**, followed by **more speech**. 

Generate the test file:

```bash
python3 -c "
from gtts import gTTS
from pydub import AudioSegment

# Generate two distinct sentences
gTTS('Hello, this is the first sentence.', lang='en').save('/tmp/part1.mp3')
gTTS('And this is the second sentence after a pause.', lang='en').save('/tmp/part2.mp3')

# Combine them with 2 seconds of absolute silence in the middle
audio = AudioSegment.from_mp3('/tmp/part1.mp3') + AudioSegment.silent(duration=2000) + AudioSegment.from_mp3('/tmp/part2.mp3')

# Export as 16kHz Mono WAV
audio.set_frame_rate(16000).set_channels(1).set_sample_width(2).export('/tmp/eou_test.wav', format='wav')
print('EoU test audio generated: /tmp/eou_test.wav')
"
```

Now, stream this audio to the server. Watch how it splits the two sentences the moment the 2-second silence triggers the EoU detector. You may also test with your own files:

```bash
python3 << 'PYEOF'
import os
import riva.client

EXTERNAL_IP = os.environ["EXTERNAL_IP"]
auth = riva.client.Auth(uri=f"{EXTERNAL_IP}:50051", use_ssl=False)
asr_service = riva.client.ASRService(auth)

config = riva.client.StreamingRecognitionConfig(
    config=riva.client.RecognitionConfig(
        encoding=riva.client.AudioEncoding.LINEAR_PCM,
        sample_rate_hertz=16000,
        audio_channel_count=1,
        language_code="en-US",
        max_alternatives=1,
        enable_automatic_punctuation=True,
    ),
    interim_results=True,
)

with open("/tmp/eou_test.wav", "rb") as f:
    f.read(44)
    chunks = iter(lambda: f.read(8000), b"")
    
    utterance_count = 1
    for resp in asr_service.streaming_response_generator(audio_chunks=chunks, streaming_config=config):
        for result in resp.results:
            text = result.alternatives[0].transcript
            if result.is_final:
                print(f"\n👉 [EOU DETECTED! Utterance {utterance_count} Finalized]: {text}\n")
                utterance_count += 1
            else:
                print(f"   [Streaming...]: {text}")
PYEOF
```

---

## Cleanup

When you're done, delete everything to stop charges (~$0.83/hr):

```bash
# Remove the Helm release
helm uninstall riva-asr

# Delete the GKE cluster
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

---

## Appendix: 

---

## Troubleshooting

### Pod stuck in `Pending`
```bash
kubectl describe pod -l app.kubernetes.io/instance=riva-asr
```
Common causes: no GPU available (check quota), insufficient CPU/memory on the node.

### Pod in `CrashLoopBackOff`
```bash
kubectl logs -l app.kubernetes.io/instance=riva-asr --previous
```
Common causes: invalid NGC_API_KEY, model download failure, insufficient GPU memory.

### LoadBalancer IP stays `<pending>`
```bash
kubectl describe svc riva-asr-riva-nim
```
Check if firewall rules are blocking. You can also use port-forwarding as a workaround:
```bash
kubectl port-forward svc/riva-asr-riva-nim 9000:9000 50051:50051
# Then use localhost instead of EXTERNAL_IP
```

### Health check returns `{"ready": false}`
The model is still loading. Check logs for TensorRT optimization progress:
```bash
kubectl logs -f -l app.kubernetes.io/instance=riva-asr
```

### Helm chart not found
```bash
helm repo update nim
helm search repo nim/riva-nim --versions
```
If empty, re-add the repo with correct NGC credentials.
