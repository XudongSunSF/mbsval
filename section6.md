# Section 6: Automatic Delta Detection (No parentRequestId)

## Overview

When a request arrives **WITHOUT** a `parentRequestId`, the system uses **automatic delta detection** to find suitable base sessions for reuse. This involves computing content-based fingerprints and searching for compatible sessions.

This document clarifies what gets fingerprinted and how base session matching works for supported delta levels (1, 3, 4).

---

## 6.A Content-Based Fingerprinting

### 6.A.1 What Gets Fingerprinted?

The system computes fingerprints on **four key dimensions**:

1. **Model Configuration Fingerprint** - Which models, which versions
2. **Economic Data Fingerprint** - Rate paths, historical data versions
3. **Portfolio Fingerprint** - Portfolio characteristics (not full data)
4. **Computation Context Fingerprint** - Global epoch, domain versions

**Combined Fingerprint Formula:**

```
requestFingerprint = H(
    modelConfigFingerprint || 
    economicDataFingerprint || 
    portfolioFingerprint || 
    contextFingerprint
)
```

Where:
- `H()` = SHA-256 hash function
- `||` = concatenation operator
- All components use canonical serialization (sorted keys, deterministic encoding)

---

### 6.A.2 Component Fingerprints in Detail

#### 6.A.2.1 Model Configuration Fingerprint

**What it includes:**
- Set of model names (sorted alphabetically)
- Model version for each model
- Model parameter file paths

**NOT included:**
- Actual model parameter values
- Model construction metadata
- Model performance metrics

**Canonical representation:**

```json
{
  "models": {
    "CAPM": {
      "version": "CAPM_v2",
      "parameterFile": "/production/models/CAPM_params_v2.json",
      "domainVersion": 5
    },
    "CFPM_2": {
      "version": "CFPM_2",
      "parameterFile": "/production/models/CFPM_2_params.json",
      "domainVersion": 5
    },
    "CRT": {
      "version": "CRT_v1",
      "parameterFile": "/production/models/CRT_params_v1.json",
      "domainVersion": 5
    },
    "JAPM": {
      "version": "JAPM_v3",
      "parameterFile": "/production/models/JAPM_params_v3.json",
      "domainVersion": 5
    },
    "PLDM": {
      "version": "PLDM_v1",
      "parameterFile": "/production/models/PLDM_params.json",
      "domainVersion": 5
    }
  }
}
```

**Fingerprint calculation:**

```cpp
std::string computeModelConfigFingerprint(const ModelConfiguration& config) {
    // Sort models alphabetically
    std::map<std::string, ModelSpec> sortedModels = config.models;
    
    // Canonical JSON serialization
    json canonical;
    canonical["schemaVersion"] = "1.0";
    canonical["models"] = json::object();
    
    for (const auto& [modelName, spec] : sortedModels) {
        canonical["models"][modelName] = {
            {"version", spec.version},
            {"parameterFile", spec.parameterFile},
            {"domainVersion", spec.domainVersion}
        };
    }
    
    // Serialize to deterministic string
    std::string canonicalStr = canonical.dump();
    
    // SHA-256 hash
    return sha256(canonicalStr).substr(0, 16);  // First 16 chars
}

// Example output: "a7f3c1e1b2d4e5f6"
```

#### 6.A.2.2 Economic Data Fingerprint

**What it includes:**
- Rate path version ID
- Historical data version ID
- Market data version IDs (spreads, volatilities, etc.)

**NOT included:**
- Actual rate values
- Full historical series
- Portfolio data (separate dimension)

**Canonical representation:**

```json
{
  "economicData": {
    "ratePaths": {
      "versionId": "3.3.a1b2c3d4.0",
      "domainVersion": 3,
      "description": "Market rates 2025-10-29"
    },
    "historicalData": {
      "versionId": "3.2.m3n4o5p6.0",
      "domainVersion": 2,
      "asOfDate": "2025-09-30"
    },
    "marketData": {
      "spreads": {
        "versionId": "3.4.s1t2u3v4.0",
        "domainVersion": 4
      }
    }
  }
}
```

**Fingerprint calculation:**

```cpp
std::string computeEconomicDataFingerprint(const EconomicData& data) {
    json canonical;
    canonical["schemaVersion"] = "1.0";
    canonical["economicData"] = {
        {"ratePaths", {
            {"versionId", data.ratePaths.versionId},
            {"domainVersion", data.ratePaths.domainVersion}
        }},
        {"historicalData", {
            {"versionId", data.historicalData.versionId},
            {"domainVersion", data.historicalData.domainVersion}
        }},
        {"marketData", {
            {"spreads", {
                {"versionId", data.marketData.spreads.versionId},
                {"domainVersion", data.marketData.spreads.domainVersion}
            }}
        }}
    };
    
    std::string canonicalStr = canonical.dump();
    return sha256(canonicalStr).substr(0, 16);
}

// Example output: "e8f9g0h1i2j3k4l5"
```

#### 6.A.2.3 Portfolio Fingerprint

**What it includes:**
- Portfolio size (number of instruments)
- Portfolio composition hash (loan IDs, sorted)
- Aggregate characteristics (average LTV, WAC, WAM)

**NOT included:**
- Full loan details (that's in portfolio data)
- Individual loan characteristics
- Valuation results

**Canonical representation:**

```json
{
  "portfolio": {
    "size": 10000,
    "compositionHash": "u1v2w3x4y5z6a7b8",
    "characteristics": {
      "averageLTV": 0.75,
      "weightedAverageCoupon": 0.062,
      "weightedAverageMaturity": 28.5
    },
    "dataVersionId": "3.1.u1v2w3x4.0"
  }
}
```

**Fingerprint calculation:**

```cpp
std::string computePortfolioFingerprint(const Portfolio& portfolio) {
    // Sort loan IDs for deterministic hashing
    std::vector<std::string> sortedLoanIds = portfolio.getLoanIds();
    std::sort(sortedLoanIds.begin(), sortedLoanIds.end());
    
    // Hash loan IDs
    std::string compositionHash = sha256(join(sortedLoanIds, ",")).substr(0, 16);
    
    json canonical;
    canonical["schemaVersion"] = "1.0";
    canonical["portfolio"] = {
        {"size", portfolio.size()},
        {"compositionHash", compositionHash},
        {"characteristics", {
            {"averageLTV", roundToDecimal(portfolio.avgLTV(), 4)},
            {"WAC", roundToDecimal(portfolio.weightedAvgCoupon(), 4)},
            {"WAM", roundToDecimal(portfolio.weightedAvgMaturity(), 2)}
        }},
        {"dataVersionId", portfolio.versionId}
    };
    
    std::string canonicalStr = canonical.dump();
    return sha256(canonicalStr).substr(0, 16);
}

// Example output: "p7q8r9s0t1u2v3w4"
```

#### 6.A.2.4 Computation Context Fingerprint

**What it includes:**
- Global epoch
- Domain versions for all domains
- System configuration version

**Canonical representation:**

```json
{
  "context": {
    "globalEpoch": 3,
    "domainVersions": {
      "modelParameters": 5,
      "ratePaths": 3,
      "historicalData": 2,
      "portfolioData": 1,
      "marketData": 4
    },
    "systemConfigVersion": "2.1.0"
  }
}
```

**Fingerprint calculation:**

```cpp
std::string computeContextFingerprint(const ComputationContext& ctx) {
    json canonical;
    canonical["schemaVersion"] = "1.0";
    canonical["context"] = {
        {"globalEpoch", ctx.globalEpoch},
        {"domainVersions", ctx.domainVersions},  // Already sorted map
        {"systemConfigVersion", ctx.systemConfigVersion}
    };
    
    std::string canonicalStr = canonical.dump();
    return sha256(canonicalStr).substr(0, 16);
}

// Example output: "c5d6e7f8g9h0i1j2"
```

---

### 6.A.3 Combined Request Fingerprint

**Complete fingerprint computation:**

```cpp
struct RequestFingerprints {
    std::string modelConfig;      // "a7f3c1e1b2d4e5f6"
    std::string economicData;     // "e8f9g0h1i2j3k4l5"
    std::string portfolio;        // "p7q8r9s0t1u2v3w4"
    std::string context;          // "c5d6e7f8g9h0i1j2"
    std::string combined;         // "x9y0z1a2b3c4d5e6"
};

RequestFingerprints computeRequestFingerprints(const ValuationRequest& request) {
    RequestFingerprints fps;
    
    // Component fingerprints
    fps.modelConfig = computeModelConfigFingerprint(request.models);
    fps.economicData = computeEconomicDataFingerprint(request.economicData);
    fps.portfolio = computePortfolioFingerprint(request.portfolio);
    fps.context = computeContextFingerprint(request.context);
    
    // Combined fingerprint
    std::string combined = fps.modelConfig + ":" + 
                          fps.economicData + ":" + 
                          fps.portfolio + ":" + 
                          fps.context;
    fps.combined = sha256(combined).substr(0, 16);
    
    return fps;
}
```

**Example output:**

```json
{
  "fingerprints": {
    "modelConfig": "a7f3c1e1b2d4e5f6",
    "economicData": "e8f9g0h1i2j3k4l5",
    "portfolio": "p7q8r9s0t1u2v3w4",
    "context": "c5d6e7f8g9h0i1j2",
    "combined": "x9y0z1a2b3c4d5e6"
  }
}
```

---

## 6.B Base Session Matching Strategy

### 6.B.1 Exact Match Search

**Goal:** Find a session with identical configuration

**Algorithm:**

```cpp
std::optional<SessionMetadata> findExactMatch(
    const RequestFingerprints& requestFps,
    const VersionRegistry& registry
) {
    // Try exact combined fingerprint match
    auto session = registry.findByFingerprint(requestFps.combined);
    if (session) {
        return session;
    }
    
    return std::nullopt;
}
```

**If exact match found:**
- ✅ Return cached results directly
- ✅ No delta processing needed
- ✅ 100% cache hit

**If no exact match:**
- ➡️ Proceed to similarity search

---

### 6.B.2 Similarity Search (Level 1: Model Version Change)

**Goal:** Find sessions with similar model configuration

**Similarity criteria for Level 1:**
- ✅ Same context (global epoch, domain versions)
- ✅ Same economic data (rate paths, historical data)
- ✅ Same portfolio
- ❓ Different model versions for SOME models

**Algorithm:**

```cpp
struct SimilarityScore {
    std::string sessionId;
    double score;              // 0.0 to 1.0
    int modelsReused;          // Count of unchanged models
    int modelsChanged;         // Count of changed models
    std::vector<std::string> changedModels;
};

std::vector<SimilarityScore> findSimilarSessions_Level1(
    const RequestFingerprints& requestFps,
    const ValuationRequest& request,
    const VersionRegistry& registry
) {
    std::vector<SimilarityScore> candidates;
    
    // Find sessions with matching context, economic data, and portfolio
    auto sessions = registry.findByFingerprints(
        requestFps.context,
        requestFps.economicData,
        requestFps.portfolio
    );
    
    for (const auto& session : sessions) {
        SimilarityScore score;
        score.sessionId = session.sessionId;
        
        // Compare model configurations
        int modelsReused = 0;
        int modelsChanged = 0;
        
        for (const auto& [modelName, requestSpec] : request.models) {
            if (session.models.count(modelName)) {
                const auto& sessionSpec = session.models.at(modelName);
                
                if (sessionSpec.versionId == requestSpec.versionId) {
                    modelsReused++;
                } else {
                    modelsChanged++;
                    score.changedModels.push_back(modelName);
                }
            } else {
                // Model not in session (model set change)
                modelsChanged++;
                score.changedModels.push_back(modelName);
            }
        }
        
        // Calculate similarity score
        int totalModels = request.models.size();
        score.modelsReused = modelsReused;
        score.modelsChanged = modelsChanged;
        score.score = static_cast<double>(modelsReused) / totalModels;
        
        // Only consider if similarity is above threshold
        if (score.score >= 0.50) {  // At least 50% model reuse
            candidates.push_back(score);
        }
    }
    
    // Sort by similarity score (descending)
    std::sort(candidates.begin(), candidates.end(),
              [](const auto& a, const auto& b) { return a.score > b.score; });
    
    return candidates;
}
```

**Example output:**

```json
{
  "candidates": [
    {
      "sessionId": "session_abc123",
      "score": 0.80,
      "modelsReused": 4,
      "modelsChanged": 1,
      "changedModels": ["CFPM_1"]
    },
    {
      "sessionId": "session_def456",
      "score": 0.60,
      "modelsReused": 3,
      "modelsChanged": 2,
      "changedModels": ["CFPM_1", "JAPM"]
    }
  ]
}
```

---

### 6.B.3 Similarity Search (Level 3: Model Set Change - Partial)

**Goal:** Find sessions with overlapping model sets

**Similarity criteria for Level 3:**
- ✅ Same context
- ✅ Same economic data
- ✅ Same portfolio
- ❓ Different model SET (some models added/removed/changed)

**Algorithm:**

```cpp
std::vector<SimilarityScore> findSimilarSessions_Level3(
    const RequestFingerprints& requestFps,
    const ValuationRequest& request,
    const VersionRegistry& registry
) {
    std::vector<SimilarityScore> candidates;
    
    // Find sessions with matching context, economic data, and portfolio
    auto sessions = registry.findByFingerprints(
        requestFps.context,
        requestFps.economicData,
        requestFps.portfolio
    );
    
    for (const auto& session : sessions) {
        SimilarityScore score;
        score.sessionId = session.sessionId;
        
        // Calculate set intersection
        std::set<std::string> requestModels;
        std::set<std::string> sessionModels;
        
        for (const auto& [name, spec] : request.models) {
            requestModels.insert(name + ":" + spec.version);
        }
        
        for (const auto& [name, spec] : session.models) {
            sessionModels.insert(name + ":" + spec.version);
        }
        
        // Find intersection (reusable models)
        std::vector<std::string> intersection;
        std::set_intersection(
            requestModels.begin(), requestModels.end(),
            sessionModels.begin(), sessionModels.end(),
            std::back_inserter(intersection)
        );
        
        score.modelsReused = intersection.size();
        score.modelsChanged = requestModels.size() - intersection.size();
        score.score = static_cast<double>(score.modelsReused) / requestModels.size();
        
        // Only consider if at least 40% overlap for model set change
        if (score.score >= 0.40) {
            candidates.push_back(score);
        }
    }
    
    // Sort by similarity score (descending)
    std::sort(candidates.begin(), candidates.end(),
              [](const auto& a, const auto& b) { return a.score > b.score; });
    
    return candidates;
}
```

---

### 6.B.4 Similarity Search (Level 4: Failed-Instrument Repricing)

**Goal:** Find sessions with same configuration but different portfolio subset

**Similarity criteria for Level 4:**
- ✅ Same context
- ✅ Same economic data
- ✅ Same model configuration
- ❓ SUPERSET of portfolio (parent has more instruments)

**Algorithm:**

```cpp
std::vector<SimilarityScore> findSimilarSessions_Level4(
    const RequestFingerprints& requestFps,
    const ValuationRequest& request,
    const VersionRegistry& registry
) {
    std::vector<SimilarityScore> candidates;
    
    // Find sessions with matching context, economic data, and models
    auto sessions = registry.findByFingerprints(
        requestFps.context,
        requestFps.economicData,
        requestFps.modelConfig
    );
    
    for (const auto& session : sessions) {
        // Check if session portfolio is a superset of request portfolio
        const auto& sessionPortfolio = session.portfolioInstruments;
        const auto& requestInstruments = request.failedInstruments;
        
        // For failed-instrument repricing:
        // Request has a SUBSET of instruments (failed ones)
        // Session should have the FULL portfolio
        
        bool isSuperset = std::all_of(
            requestInstruments.begin(),
            requestInstruments.end(),
            [&](const std::string& instrument) {
                return sessionPortfolio.count(instrument) > 0;
            }
        );
        
        if (isSuperset) {
            SimilarityScore score;
            score.sessionId = session.sessionId;
            score.modelsReused = request.models.size();  // All models reused
            score.modelsChanged = 0;
            score.score = 1.0;  // Perfect match for models
            
            candidates.push_back(score);
        }
    }
    
    return candidates;
}
```

---

## 6.C Complete Automatic Delta Detection Workflow

### 6.C.1 High-Level Algorithm

```cpp
struct DeltaDecision {
    enum Type {
        EXACT_MATCH,           // Use cached results directly
        DELTA_PROCESSING,      // Use delta path
        FULL_REBUILD           // No suitable base found
    };
    
    Type type;
    std::optional<std::string> baseSessionId;
    double reuseRatio;
    std::string reason;
};

DeltaDecision automaticDeltaDetection(
    const ValuationRequest& request,
    const VersionRegistry& registry
) {
    // Step 1: Compute fingerprints
    RequestFingerprints fps = computeRequestFingerprints(request);
    
    // Step 2: Try exact match
    auto exactMatch = findExactMatch(fps, registry);
    if (exactMatch) {
        return {
            .type = DeltaDecision::EXACT_MATCH,
            .baseSessionId = exactMatch->sessionId,
            .reuseRatio = 1.0,
            .reason = "Exact fingerprint match found"
        };
    }
    
    // Step 3: Try similarity search (multi-level)
    std::vector<SimilarityScore> candidates;
    
    // Level 1: Model version change
    auto level1Candidates = findSimilarSessions_Level1(fps, request, registry);
    candidates.insert(candidates.end(), level1Candidates.begin(), level1Candidates.end());
    
    // Level 3: Model set change
    auto level3Candidates = findSimilarSessions_Level3(fps, request, registry);
    candidates.insert(candidates.end(), level3Candidates.begin(), level3Candidates.end());
    
    // Level 4: Failed-instrument repricing
    if (request.failedInstruments.size() > 0) {
        auto level4Candidates = findSimilarSessions_Level4(fps, request, registry);
        candidates.insert(candidates.end(), level4Candidates.begin(), level4Candidates.end());
    }
    
    // Step 4: Select best candidate
    if (candidates.empty()) {
        return {
            .type = DeltaDecision::FULL_REBUILD,
            .baseSessionId = std::nullopt,
            .reuseRatio = 0.0,
            .reason = "No suitable base session found"
        };
    }
    
    // Sort by score (already sorted in each level, but merge here)
    std::sort(candidates.begin(), candidates.end(),
              [](const auto& a, const auto& b) { return a.score > b.score; });
    
    const auto& bestCandidate = candidates[0];
    
    // Step 5: Evaluate delta threshold (default 20%)
    const double DELTA_THRESHOLD = 0.20;
    double changeRatio = 1.0 - bestCandidate.score;
    
    if (changeRatio <= DELTA_THRESHOLD) {
        return {
            .type = DeltaDecision::DELTA_PROCESSING,
            .baseSessionId = bestCandidate.sessionId,
            .reuseRatio = bestCandidate.score,
            .reason = "Delta processing acceptable (change ratio: " + 
                     std::to_string(changeRatio) + " <= " + 
                     std::to_string(DELTA_THRESHOLD) + ")"
        };
    } else {
        return {
            .type = DeltaDecision::FULL_REBUILD,
            .baseSessionId = std::nullopt,
            .reuseRatio = 0.0,
            .reason = "Delta change ratio too high (change ratio: " + 
                     std::to_string(changeRatio) + " > " + 
                     std::to_string(DELTA_THRESHOLD) + ")"
        };
    }
}
```

---

## 6.D Concrete Examples

### Example 1: Exact Match (No Delta Needed)

**Request A:**
```json
{
  "requestId": "req_001",
  "models": {
    "CAPM": {"version": "CAPM_v2", "parameterFile": "/production/models/CAPM_params_v2.json"},
    "CFPM_2": {"version": "CFPM_2", "parameterFile": "/production/models/CFPM_2_params.json"},
    "JAPM": {"version": "JAPM_v3", "parameterFile": "/production/models/JAPM_params_v3.json"},
    "PLDM": {"version": "PLDM_v1", "parameterFile": "/production/models/PLDM_params.json"},
    "CRT": {"version": "CRT_v1", "parameterFile": "/production/models/CRT_params_v1.json"}
  },
  "economicData": {
    "ratePaths": {"versionId": "3.3.a1b2c3d4.0"},
    "historicalData": {"versionId": "3.2.m3n4o5p6.0"}
  },
  "portfolio": {
    "versionId": "3.1.u1v2w3x4.0",
    "size": 10000
  },
  "context": {
    "globalEpoch": 3,
    "domainVersions": {"modelParameters": 5, "ratePaths": 3, "historicalData": 2}
  }
}
```

**Fingerprints:**
```json
{
  "modelConfig": "a7f3c1e1b2d4e5f6",
  "economicData": "e8f9g0h1i2j3k4l5",
  "portfolio": "p7q8r9s0t1u2v3w4",
  "context": "c5d6e7f8g9h0i1j2",
  "combined": "x9y0z1a2b3c4d5e6"
}
```

**Request B (5 minutes later):**
Same configuration as Request A

**Fingerprints:**
```json
{
  "combined": "x9y0z1a2b3c4d5e6"  // ✅ EXACT MATCH!
}
```

**Delta Decision:**
```json
{
  "type": "EXACT_MATCH",
  "baseSessionId": "session_req_001",
  "reuseRatio": 1.0,
  "reason": "Exact fingerprint match found",
  "action": "Return cached results directly, no computation needed"
}
```

---

### Example 2: Level 1 Delta (Model Version Change)

**Request A (Base):**
```json
{
  "requestId": "req_002",
  "models": {
    "CAPM": {"version": "CAPM_v2", "versionId": "3.5.8a9f2e1b.0"},
    "CFPM_1": {"version": "CFPM_1", "versionId": "3.5.a7f3c1e1.0"},  // ← Old version
    "JAPM": {"version": "JAPM_v3", "versionId": "3.5.c4e5f789.0"},
    "PLDM": {"version": "PLDM_v1", "versionId": "3.5.d1b2c3a4.0"},
    "CRT": {"version": "CRT_v1", "versionId": "3.5.e7f8a9b0.0"}
  },
  "economicData": {
    "ratePaths": {"versionId": "3.3.a1b2c3d4.0"},
    "historicalData": {"versionId": "3.2.m3n4o5p6.0"}
  },
  "portfolio": {"versionId": "3.1.u1v2w3x4.0"}
}
```

**Fingerprints for Request A:**
```json
{
  "modelConfig": "m1n2o3p4q5r6s7t8",      // Hash includes CFPM_1
  "economicData": "e8f9g0h1i2j3k4l5",
  "portfolio": "p7q8r9s0t1u2v3w4",
  "context": "c5d6e7f8g9h0i1j2",
  "combined": "z1a2b3c4d5e6f7g8"
}
```

**Request B (Delta Request):**
```json
{
  "requestId": "req_003",
  "models": {
    "CAPM": {"version": "CAPM_v2", "versionId": "3.5.8a9f2e1b.0"},
    "CFPM_2": {"version": "CFPM_2", "versionId": "3.5.f4ddb0c2.0"},  // ← NEW version
    "JAPM": {"version": "JAPM_v3", "versionId": "3.5.c4e5f789.0"},
    "PLDM": {"version": "PLDM_v1", "versionId": "3.5.d1b2c3a4.0"},
    "CRT": {"version": "CRT_v1", "versionId": "3.5.e7f8a9b0.0"}
  },
  "economicData": {
    "ratePaths": {"versionId": "3.3.a1b2c3d4.0"},      // Same
    "historicalData": {"versionId": "3.2.m3n4o5p6.0"}  // Same
  },
  "portfolio": {"versionId": "3.1.u1v2w3x4.0"}         // Same
}
```

**Fingerprints for Request B:**
```json
{
  "modelConfig": "h9i0j1k2l3m4n5o6",      // Different (CFPM_2 instead of CFPM_1)
  "economicData": "e8f9g0h1i2j3k4l5",     // ✅ Same
  "portfolio": "p7q8r9s0t1u2v3w4",        // ✅ Same
  "context": "c5d6e7f8g9h0i1j2",          // ✅ Same
  "combined": "w7x8y9z0a1b2c3d4"          // Different
}
```

**Similarity Search:**
```json
{
  "exactMatch": null,
  "similarSessions": [
    {
      "sessionId": "session_req_002",
      "score": 0.80,
      "modelsReused": 4,
      "modelsChanged": 1,
      "changedModels": ["CFPM_1"],
      "matchingFingerprints": {
        "economicData": true,
        "portfolio": true,
        "context": true,
        "modelConfig": false
      }
    }
  ]
}
```

**Delta Decision:**
```json
{
  "type": "DELTA_PROCESSING",
  "baseSessionId": "session_req_002",
  "reuseRatio": 0.80,
  "changeRatio": 0.20,
  "reason": "Level 1 delta - model version change (CFPM_1 → CFPM_2)",
  "action": "Reuse 4 models from session_req_002, construct 1 new model (CFPM_2)",
  "estimatedTimeSaving": "80%"
}
```

---

### Example 3: Level 3 Delta (Model Set Change)

**Request A (Base):**
```json
{
  "requestId": "req_004",
  "models": {
    "CAPM": {"versionId": "3.5.8a9f2e1b.0"},
    "CFPM_1": {"versionId": "3.5.a7f3c1e1.0"},
    "PLDM": {"versionId": "3.5.d1b2c3a4.0"}
    // Only 3 models
  },
  "economicData": {"ratePaths": {"versionId": "3.3.a1b2c3d4.0"}},
  "portfolio": {"versionId": "3.1.u1v2w3x4.0"}
}
```

**Request B (Different Model Set):**
```json
{
  "requestId": "req_005",
  "models": {
    "CAPM": {"versionId": "3.5.8a9f2e1b.0"},      // ✅ Same
    "JFPM": {"versionId": "3.5.9e8f7a6b.0"},      // ← New model
    "CRT": {"versionId": "3.5.e7f8a9b0.0"}        // ← New model
    // CFPM_1 removed, PLDM removed, JFPM and CRT added
  },
  "economicData": {"ratePaths": {"versionId": "3.3.a1b2c3d4.0"}},  // Same
  "portfolio": {"versionId": "3.1.u1v2w3x4.0"}                      // Same
}
```

**Similarity Search:**
```json
{
  "similarSessions": [
    {
      "sessionId": "session_req_004",
      "score": 0.33,
      "modelsReused": 1,       // Only CAPM reused
      "modelsChanged": 2,      // JFPM and CRT new
      "reuseRatio": 0.33,
      "changeRatio": 0.67
    }
  ]
}
```

**Delta Decision:**
```json
{
  "type": "FULL_REBUILD",
  "baseSessionId": null,
  "reason": "Change ratio (0.67) exceeds threshold (0.20)",
  "action": "Full rebuild - construct all 3 models from scratch",
  "note": "Only 33% reuse is below minimum threshold for delta processing"
}
```

**Alternative with higher reuse:**

If Request B had 4 models with 3 in common:

```json
{
  "models": {
    "CAPM": {"versionId": "3.5.8a9f2e1b.0"},   // ✅ From base
    "CFPM_1": {"versionId": "3.5.a7f3c1e1.0"}, // ✅ From base
    "PLDM": {"versionId": "3.5.d1b2c3a4.0"},   // ✅ From base
    "JFPM": {"versionId": "3.5.9e8f7a6b.0"}    // ← New
  }
}
```

Then:
```json
{
  "score": 0.75,
  "modelsReused": 3,
  "modelsChanged": 1,
  "changeRatio": 0.25,  // Still above 0.20 threshold!
  
  "decision": {
    "type": "FULL_REBUILD",
    "reason": "Change ratio (0.25) exceeds threshold (0.20) by small margin, but cost model evaluation showed full rebuild is cheaper"
  }
}
```

---

### Example 4: Level 4 Delta (Failed-Instrument Repricing)

**Request A (Base - Full Portfolio):**
```json
{
  "requestId": "req_006",
  "models": {
    "CAPM": {"versionId": "3.5.8a9f2e1b.0"},
    "CFPM_2": {"versionId": "3.5.f4ddb0c2.0"},
    "JAPM": {"versionId": "3.5.c4e5f789.0"},
    "PLDM": {"versionId": "3.5.d1b2c3a4.0"},
    "CRT": {"versionId": "3.5.e7f8a9b0.0"}
  },
  "portfolio": {
    "versionId": "3.1.u1v2w3x4.0",
    "instruments": ["LOAN_00001", "LOAN_00002", ..., "LOAN_10000"],
    "size": 10000
  },
  "results": {
    "successful": 9998,
    "failed": ["LOAN_12345", "LOAN_67890"]
  }
}
```

**Fingerprints:**
```json
{
  "modelConfig": "a7f3c1e1b2d4e5f6",
  "economicData": "e8f9g0h1i2j3k4l5",
  "portfolio": "p7q8r9s0t1u2v3w4",      // Full portfolio
  "context": "c5d6e7f8g9h0i1j2"
}
```

**Request B (Failed Instruments Only):**
```json
{
  "requestId": "req_007",
  "models": {
    "CAPM": {"versionId": "3.5.8a9f2e1b.0"},     // ✅ Same
    "CFPM_2": {"versionId": "3.5.f4ddb0c2.0"},   // ✅ Same
    "JAPM": {"versionId": "3.5.c4e5f789.0"},     // ✅ Same
    "PLDM": {"versionId": "3.5.d1b2c3a4.0"},     // ✅ Same
    "CRT": {"versionId": "3.5.e7f8a9b0.0"}       // ✅ Same
  },
  "economicData": {
    "ratePaths": {"versionId": "3.3.a1b2c3d4.0"},     // ✅ Same
    "historicalData": {"versionId": "3.2.m3n4o5p6.0"} // ✅ Same
  },
  "portfolio": {
    "versionId": "3.1.y5z6a7b8.0",                     // Different (subset)
    "instruments": ["LOAN_12345", "LOAN_67890"],       // Only failed loans
    "size": 2,
    "isSubsetOf": "3.1.u1v2w3x4.0",                    // Reference to parent
    "parentRequest": "req_006"                          // Optional hint
  },
  "operationType": "FAILED_INSTRUMENT_REPRICING"
}
```

**Fingerprints:**
```json
{
  "modelConfig": "a7f3c1e1b2d4e5f6",     // ✅ Same as Request A
  "economicData": "e8f9g0h1i2j3k4l5",    // ✅ Same as Request A
  "portfolio": "f9g0h1i2j3k4l5m6",       // Different (subset fingerprint)
  "context": "c5d6e7f8g9h0i1j2"          // ✅ Same as Request A
}
```

**Similarity Search (Level 4 Specific):**
```cpp
// Check if Request B portfolio is a subset of any existing session
auto parentSession = findSuperset(request.portfolio.instruments, registry);

if (parentSession && 
    parentSession.modelConfig == request.modelConfig &&
    parentSession.economicData == request.economicData) {
    
    // Found suitable parent for failed-instrument repricing
    return {
        .sessionId = parentSession.sessionId,
        .score = 1.0,  // Perfect model and data match
        .instrumentsToRecompute = request.portfolio.size(),
        .instrumentsToReuse = parentSession.portfolio.size() - request.portfolio.size()
    };
}
```

**Delta Decision:**
```json
{
  "type": "DELTA_PROCESSING",
  "deltaLevel": "LEVEL_4_FAILED_INSTRUMENTS",
  "baseSessionId": "session_req_006",
  "reuseRatio": 0.9998,
  "action": {
    "reuseResults": {
      "count": 9998,
      "source": "session_req_006"
    },
    "recomputeInstruments": {
      "count": 2,
      "instruments": ["LOAN_12345", "LOAN_67890"]
    },
    "mergeStrategy": "Replace failed results with new computations"
  },
  "estimatedTimeSaving": "99.98%",
  "reason": "Level 4 delta - failed-instrument repricing"
}
```

---

## 6.E Registry Data Structures

### 6.E.1 Session Metadata in Version Registry

```cpp
struct SessionMetadata {
    std::string sessionId;
    std::string requestId;
    
    // Fingerprints
    std::string modelConfigFingerprint;
    std::string economicDataFingerprint;
    std::string portfolioFingerprint;
    std::string contextFingerprint;
    std::string combinedFingerprint;
    
    // Models
    std::map<std::string, ModelSpec> models;
    
    // Economic data
    EconomicDataSpec economicData;
    
    // Portfolio
    PortfolioSpec portfolio;
    std::set<std::string> portfolioInstruments;  // For subset matching
    
    // Context
    int globalEpoch;
    std::map<std::string, int> domainVersions;
    
    // Metadata
    std::string userId;
    std::chrono::system_clock::time_point timestamp;
    bool isActive;
    
    // Results tracking
    int successfulResults;
    std::vector<std::string> failedInstruments;
};
```

### 6.E.2 Registry Indexes

```cpp
class VersionRegistry {
private:
    // Primary index: sessionId → SessionMetadata
    std::unordered_map<std::string, SessionMetadata> sessions;
    
    // Secondary indexes for fast lookup
    std::unordered_map<std::string, std::vector<std::string>> 
        fingerprintIndex_combined;
    
    std::unordered_map<std::string, std::vector<std::string>>
        fingerprintIndex_modelConfig;
        
    std::unordered_map<std::string, std::vector<std::string>>
        fingerprintIndex_economicData;
        
    std::unordered_map<std::string, std::vector<std::string>>
        fingerprintIndex_portfolio;
        
    std::unordered_map<std::string, std::vector<std::string>>
        fingerprintIndex_context;
    
public:
    // Exact match
    std::optional<SessionMetadata> findByFingerprint(
        const std::string& combinedFingerprint
    );
    
    // Multi-dimensional search
    std::vector<SessionMetadata> findByFingerprints(
        const std::string& contextFp,
        const std::string& economicDataFp,
        const std::string& portfolioFp
    );
    
    // Subset search (for Level 4)
    std::optional<SessionMetadata> findSuperset(
        const std::set<std::string>& instruments
    );
};
```

---

## Summary

### Fingerprinting Strategy

| Component | What Gets Fingerprinted | NOT Fingerprinted |
|-----------|------------------------|-------------------|
| **Models** | Model names, versions, parameter file paths | Actual parameter values, construction metadata |
| **Economic Data** | Version IDs (rates, historical, market data) | Actual rate values, full time series |
| **Portfolio** | Size, composition hash, aggregate characteristics | Individual loan details |
| **Context** | Global epoch, domain versions, system config | Runtime state, execution environment |

### Matching Strategy by Delta Level

| Level | Matching Criteria | Threshold | Action |
|-------|------------------|-----------|--------|
| **Exact Match** | All fingerprints identical | 100% | Return cached results |
| **Level 1** | Models differ, rest same | ≥50% model reuse, ≤20% change | Reuse unchanged models |
| **Level 3** | Model set differs, rest same | ≥40% overlap, evaluate cost | Reuse overlapping models |
| **Level 4** | Portfolio subset, models+data same | 100% model reuse | Reuse successful results |
| **No Match** | No suitable base found | N/A | Full rebuild |

### Key Design Principles

1. **Canonical Serialization:** All fingerprints use deterministic, sorted serialization
2. **Component Independence:** Each fingerprint computed independently for flexible matching
3. **Multi-Level Search:** Try multiple similarity strategies in priority order
4. **Cost-Based Decision:** Evaluate delta vs. rebuild cost even with matches
5. **Registry Indexing:** Multiple indexes for fast multi-dimensional lookup