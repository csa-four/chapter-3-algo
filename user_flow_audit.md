# User Verification Live Execution Flow Audit

This document traces the actual end-to-end user journey during a live Signature Verification. It maps every frontend interaction seamlessly to the backend AI and Blockchain pipelines, allowing panelists or future developers to follow the exact execution path.

## 1. Document Upload & Metadata (Frontend)
The user begins by dragging and dropping a document and setting metadata.
*   **Select Document (Upload):** Handled by `handleFiles()` and an HTML file dropzone.
    *   **File:** `client/components/verification/user-verify-flow.tsx`
    *   **Lines:** ~108-115
*   **Set Document Date:** Handled by a date picker input.
    *   **File:** `client/components/verification/user-verify-flow.tsx`
    *   **Lines:** ~138-144

## 2. Trigger Extraction Pipeline
The user clicks "Proceed to Analysis", which passes the file to the backend to run YOLO and GAN.
*   **Trigger Action:** `onDateSubmit()` calls the pipeline hook.
    *   **File:** `client/components/verification/user-verify-flow.tsx`
    *   **Line:** ~68
*   **Frontend API Call:** The pipeline hook sends a `multipart/form-data` POST request.
    *   **File:** `client/hooks/use-pipeline.ts`
    *   **Lines:** ~35-39 (`fetch("http://127.0.0.1:8008/api/extract-signatures")`)
*   **Backend Endpoint (AI Preprocessing):** Receives the document, runs YOLO to detect the signature, and GAN to erase noise.
    *   **File:** `server/app/api/endpoints.py`
    *   **Lines:** ~21-77 (`@router.post("/extract-signatures")`)

## 3. Signatory Mapping & AI Verification
The system pauses to let the user review the AI's cropped signature and assign it to a known reference in the system.
*   **Assign Reference Signatory:** User selects a target profile from a list.
    *   **File:** `client/components/verification/user-verify-flow.tsx`
    *   **Lines:** ~281-318 (`setSignatureMappings`)
*   **Start Signature Check (Trigger AI):** User clicks to execute CapsNet/Mahalanobis.
    *   **File:** `client/components/verification/user-verify-flow.tsx`
    *   **Line:** ~326 (`resumePipeline()`)
*   **Frontend API Call:** 
    *   **File:** `client/hooks/use-pipeline.ts`
    *   **Lines:** ~114-120 (`fetch("http://127.0.0.1:8008/api/verifications/verify")`)
*   **Backend Endpoint (AI Verification):** Looks up the profile, runs the CapsNet embedding, computes the Mahalanobis distance, and saves the PostgreSQL database record.
    *   **File:** `server/app/api/verification.py`
    *   **Lines:** ~89-154 (`@router.post("/verify")`)

## 4. Cryptographic Hashing & Blockchain Auditing
After the AI generates a verdict, the frontend orchestrates securely logging the results to the Ethereum blockchain.
*   **Prepare Blockchain Payloads:** Generates JSON configurations for IPFS.
    *   **File:** `client/hooks/use-pipeline.ts`
    *   **Lines:** ~148-175
*   **Generate Merkle Root & Smart Contract TX:** Calculates $H_{root}$ and executes `contract.logVerificationBatch`.
    *   **File:** `client/lib/blockchain.ts`
    *   **Lines:** ~136-153
*   **Update Backend Database with TX Hash:** Patches the backend record so the Blockchain receipt is saved in Postgres.
    *   **File:** `client/hooks/use-pipeline.ts`
    *   **Lines:** ~177-190 (`fetch("/api/verifications/{id}/blockchain", { method: "PATCH" })`)

## 5. Final Result Presentation
*   **Display Results to User:** Renders the final verdict, confidence score, IPFS CID, and Blockchain TX Hash.
    *   **File:** `client/components/verification/user-verify-flow.tsx`
    *   **Lines:** ~343-344 (Renders `<VerificationResults />` component)
