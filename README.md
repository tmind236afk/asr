This README combines the deployment guides for the **NVIDIA Parakeet** ASR models on Google Cloud, covering both serverless-style deployment (Vertex AI) and container-orchestrated deployment (GKE).

---

# NVIDIA Parakeet ASR on Google Cloud

This repository provides step-by-step guides and automation for deploying NVIDIA's Parakeet Automatic Speech Recognition (ASR) models on **Google Cloud Platform**. It covers two primary deployment architectures:

## 🚀 Deployment Options

### 1. Vertex AI (Fast & Managed)

Ideal for developers who want a managed endpoint with automatic scaling.

* **Model:** Parakeet Realtime EOU 120M.
* **Features:** End-of-Utterance (EOU) detection, base64 audio support, Flask-based serving.
* **Compute:** NVIDIA T4 GPU.
* **Guide:** `deploy_parakeet_eou_vertex_ai.ipynb` (Jupyter Notebook).

### 2. GKE with NVIDIA Riva NIM (Scale & Performance)

Ideal for production-grade streaming applications requiring high throughput and low latency.

* **Model:** Parakeet 1.1B CTC (via NVIDIA Riva NIM).
* **Features:** gRPC streaming, WebSocket Realtime API, built-in VAD (Voice Activity Detection).
* **Compute:** NVIDIA L4 GPU (G2 Standard instances).
* **Guide:** `CLOUDSHELL_DEPLOY.md` (Cloud Shell commands).

---

## 🛠️ Quick Start Requirements

Before starting either guide, ensure you have:

1. A **Google Cloud Project** with billing enabled.
2. An **NVIDIA NGC Account** and API Key (required for GKE/Riva).
3. Enabled APIs: `Compute Engine`, `Artifact Registry`, `Cloud Build`, and `Vertex AI`.

---

## 📂 File Structure

| File | Description |
| --- | --- |
| `deploy_parakeet_eou_vertex_ai.ipynb` | A Colab-ready notebook to build a custom container and deploy to a Vertex AI Endpoint. |
| `RIVA_DEPLOY.md` | A markdown guide for deploying Riva NIM on GKE using Helm charts. |

---

## 🧹 Cleanup

To avoid unnecessary charges, remember to delete your resources after testing:

* **Vertex AI:** Delete the Endpoint and Model via the notebook or Cloud Console.
* **GKE:** Run `gcloud container clusters delete [CLUSTER_NAME]` to remove the cluster and attached GPUs.

---

### ⚠️ Disclaimer

**This project is provided as a reference example only.** The code and configurations are intended to demonstrate the deployment of NVIDIA models on Google Cloud and have not been hardened for production environments.

Before using this in a production setting, please perform a comprehensive review of:

* **Security:** IAM roles, VPC service controls, and secret management.
* **Costs:** Monitor GPU utilization and set up billing alerts to avoid unexpected charges.
* **Licensing:** Ensure compliance with the [NVIDIA Open Model License](https://www.google.com/search?q=https://hf.co/nvidia/parakeet_realtime_eou_120m-v1/blob/main/LICENSE.md).

**Use at your own risk.** The authors and contributors assume no liability for any direct or indirect damages, data loss, or costs incurred through the use of this repository.

---
