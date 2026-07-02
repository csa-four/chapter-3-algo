# Authenticate Document Activity Diagram Audit

This document presents a technical audit verifying the codebase against the provided `Authenticate Document` Activity Diagram. It maps each visual activity to its exact implementation file and line of code, and documents any architectural deviations.

## Swimlane 1: Document Verifier

| Diagram Activity | Implementation File | Class / Function | Approx. Line Numbers |
| :--- | :--- | :--- | :--- |
| **Upload Scanned Document** | `server/app/api/endpoints.py` | `extract_signatures()` | ~21-46 |
| **Input Document Creation Date** | (Frontend / UI) | `UploadForm` component | N/A |
| **Display Final Verification Result** | (Frontend / UI) | `VerificationResults` component | N/A |

---

## Swimlane 2: Forensic Preprocessing Unit

| Diagram Activity | Implementation File | Class / Function | Approx. Line Numbers |
| :--- | :--- | :--- | :--- |
| **Load Raw Scan into Memory** | `server/app/services/gan_service.py` | `denoise() -> cv2.imread` | ~42 |
| **Generate Clean Signature** | `models/gan/inference/forensic_denoiser.py`| `ForensicDenoiser._run_pix2pix()` | ~20-31 |
| **Evaluate Structural Integrity** | `models/gan/inference/forensic_denoiser.py`| `ForensicDenoiser.denoise()` (Connected Components analysis) | ~59-123 |
| **Output Isolated Signature** | `models/gan/inference/forensic_denoiser.py`| `ForensicDenoiser.denoise() -> return final_output` | ~126-137 |

> [!WARNING]
> **Deviation Detected: "Is Loss > Threshold" Loop**
> The activity diagram depicts a loop where the system checks if "Loss > Threshold" during authentication (inference) and loops back to "Generate Clean Signature". 
> **Actual Implementation:** Kumpirma's AI inference engine uses a *single* deterministic Generator forward pass, followed by advanced OpenCV Connected Components analysis to evaluate structural integrity and erase hallucinations. It is incredibly efficient and does *not* recursively loop back through the GAN based on a loss threshold during live verification. 

---

## Swimlane 3: AI Verification Engine

| Diagram Activity | Implementation File | Class / Function | Approx. Line Numbers |
| :--- | :--- | :--- | :--- |
| **Create Inverted Negative Channel** | `models/capsnet/inference/verify.py` | `_load_dual_channel() -> inverted = -gray` | ~47-48 |
| **Create Grayscale Channel** | `models/capsnet/inference/verify.py` | `_load_dual_channel() -> gray = ...` | ~46-48 |
| **Initialize Coupling Coefficients** | `models/capsnet/architecture/digit_caps.py`| `DigitCaps.forward() -> log_prior_coupling = torch.zeros(...)` | ~63-65 |
| **Compute Prediction Vectors** | `models/capsnet/architecture/digit_caps.py`| `DigitCaps.forward() -> predicted_vectors = torch.einsum(...)` | ~58-59 |
| **Apply Routing-by-Agreement** | `models/capsnet/architecture/digit_caps.py`| `DigitCaps.forward()` (The internal `a_ij` loop) | ~75-76 |
| **Apply Squashing Function** | `models/capsnet/architecture/digit_caps.py`| `DigitCaps._squash()` | ~83-91 |
| **Iterations < R** | `models/capsnet/architecture/digit_caps.py`| `DigitCaps.forward() -> for i in range(self.num_iterations):`| ~68 |
| **Generate Latent Feature Vector** | `models/capsnet/inference/verify.py` | `run_verification() -> emb = out.view(-1).cpu().numpy()` | ~205-206 |

---

## Swimlane 4: Decision Logic Layer

| Diagram Activity | Implementation File | Class / Function | Approx. Line Numbers |
| :--- | :--- | :--- | :--- |
| **Calculate Mahalanobis Distance** | `models/capsnet/architecture/mahalanobis_verifier.py`| `MahalanobisVerifier.distance()` | ~145-158 |
| **$d_M < \text{Threshold}_{auth}$** | `models/capsnet/inference/verify.py` | `run_verification() -> if distance < threshold:` | ~219-228 |
| **Assign Verdict = "Genuine"/"Forged"**| `models/capsnet/inference/verify.py` | `run_verification() -> is_genuine = True/False` | ~219-228 |

> [!WARNING]
> **Deviation Detected: "Query Blockchain for Active $V_{reference}$"**
> The activity diagram indicates that the system pulls the reference parameters dynamically from the Blockchain right before calculating Mahalanobis distance.
> **Actual Implementation:** The live verification engine (`verify.py`) pulls the learned $V_{reference}$ statistics (`mean_vector` and `inverse_covariance`) from a fast, locally persisted state file (`mahalanobis_verifier.pkl`), not directly from a smart contract query. The blockchain is used to immutably anchor the *result* (Merkle Root and IPFS CID) after verification concludes, ensuring auditing integrity, rather than acting as a live NoSQL database during the split-second distance calculation. 
