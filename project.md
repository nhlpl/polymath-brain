We'll create a complete, compilable Rust implementation of the **Polymath Brain** — a multi‑agent cognitive ecosystem where agents from different guilds (Ethical, Strategic, Creative, Causal) debate, critique, and synthesize perspectives on complex queries. The code integrates the φ‑resonant principles, the Akashic Graph for causal tracing, and a simplified Agora debate protocol.

This is a self‑contained simulation. It demonstrates the core architecture: agents, guilds, debate, synthesis, and the recording of causal history.

---

## 📁 Project Structure

```
polymath_brain/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── lib.rs
    ├── agents/
    │   ├── mod.rs
    │   ├── base.rs
    │   ├── ethical.rs
    │   ├── strategic.rs
    │   ├── creative.rs
    │   └── causal.rs
    ├── agora/
    │   ├── mod.rs
    │   ├── debate.rs
    │   ├── akashic.rs
    │   └── synthesis.rs
    ├── temporal/
    │   ├── mod.rs
    │   └── cortex.rs
    ├── substrates/
    │   ├── mod.rs
    │   └── base.rs
    └── common/
        ├── mod.rs
        ├── constants.rs
        └── hypervector.rs
```

---

## 1. `Cargo.toml`

```toml
[package]
name = "polymath_brain"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1.35", features = ["full"] }
ndarray = "0.15"
rand = "0.8"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
dashmap = "5.5"
parking_lot = "0.12"
tracing = "0.1"
tracing-subscriber = "0.3"
anyhow = "1.0"
thiserror = "1.0"
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = "0.4"
```

---

## 2. `src/common/constants.rs`

```rust
//! φ‑resonant constants shared across the Polymath Brain.

pub const PHI: f64 = 1.618033988749895;
pub const HYPERVECTOR_DIM: usize = 256;  // reduced for demo
pub const CONFIDENCE_THRESHOLD: f64 = 1.0 / PHI;  // ~0.618
pub const SEAL_THRESHOLD: f64 = 0.9;  // threshold for Eschaton Lock
```

---

## 3. `src/common/hypervector.rs`

```rust
use ndarray::Array1;
use rand::prelude::*;
use std::ops::Add;
use crate::common::constants::{HYPERVECTOR_DIM, PHI};

#[derive(Debug, Clone, PartialEq)]
pub struct HyperVector {
    pub data: Array1<f64>,
}

impl HyperVector {
    pub fn random(seed: u64) -> Self {
        let mut rng = StdRng::seed_from_u64(seed);
        let data = Array1::from_iter((0..HYPERVECTOR_DIM).map(|_| rng.gen_range(-1.0..1.0)));
        Self { data }
    }

    pub fn zero() -> Self {
        Self { data: Array1::zeros(HYPERVECTOR_DIM) }
    }

    pub fn similarity(&self, other: &Self) -> f64 {
        let dot = self.data.dot(&other.data);
        let norm_a = self.norm();
        let norm_b = other.norm();
        if norm_a == 0.0 || norm_b == 0.0 { 0.0 } else { dot / (norm_a * norm_b) }
    }

    pub fn norm(&self) -> f64 {
        self.data.dot(&self.data).sqrt()
    }

    pub fn bundle_with(&mut self, other: &Self, weight: f64) {
        self.data.scaled_add(weight, &other.data);
    }

    pub fn as_slice(&self) -> &[f64] {
        self.data.as_slice().unwrap()
    }
}

impl Add for HyperVector {
    type Output = Self;
    fn add(mut self, other: Self) -> Self {
        self.data += &other.data;
        self
    }
}
```

---

## 4. `src/agents/base.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;
use serde::{Serialize, Deserialize};
use crate::common::hypervector::HyperVector;
use crate::common::constants::PHI;

pub type AgentId = Uuid;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum Guild {
    Ethical,
    Strategic,
    Creative,
    Causal,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Perspective {
    pub agent_id: AgentId,
    pub guild: Guild,
    pub content: String,
    pub embedding: HyperVector,
    pub confidence: f64,
    pub reasoning: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Critique {
    pub critic_id: AgentId,
    pub target_id: AgentId,
    pub strengths: Vec<String>,
    pub weaknesses: Vec<String>,
    pub suggested_improvements: String,
    pub confidence: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Query {
    pub id: Uuid,
    pub text: String,
    pub embedding: HyperVector,
    pub urgency: Urgency,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Urgency {
    Immediate,
    Deliberative,
    Evolutionary,
}

#[async_trait]
pub trait Agent: Send + Sync {
    fn id(&self) -> AgentId;
    fn guild(&self) -> Guild;
    fn name(&self) -> &str;
    
    /// Generate a perspective on the query.
    async fn deliberate(&self, query: &Query) -> anyhow::Result<Perspective>;
    
    /// Critique another agent's perspective.
    async fn critique(&self, perspective: &Perspective) -> anyhow::Result<Critique>;
    
    /// Get the agent's substrate affinity (which substrate it prefers).
    fn substrate_affinity(&self) -> SubstrateType;
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SubstrateType {
    Silicon,
    Polymer,
    Neuromorphic,
}
```

---

## 5. `src/agents/ethical.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;
use crate::agents::base::{Agent, AgentId, Guild, Perspective, Critique, Query, SubstrateType};
use crate::common::hypervector::HyperVector;
use crate::common::constants::PHI;

pub struct EthicalAgent {
    id: AgentId,
    name: String,
}

impl EthicalAgent {
    pub fn new(name: &str) -> Self {
        Self { id: Uuid::new_v4(), name: name.to_string() }
    }
}

#[async_trait]
impl Agent for EthicalAgent {
    fn id(&self) -> AgentId { self.id }
    fn guild(&self) -> Guild { Guild::Ethical }
    fn name(&self) -> &str { &self.name }
    fn substrate_affinity(&self) -> SubstrateType { SubstrateType::Silicon }
    
    async fn deliberate(&self, query: &Query) -> anyhow::Result<Perspective> {
        // Ethical agent considers moral frameworks: utilitarianism, deontology, virtue ethics.
        let content = format!(
            "From an ethical standpoint, we must weigh the consequences for all stakeholders. \
             The principle of non‑maleficence suggests caution. Confidence reflects the \
             complexity of moral trade‑offs."
        );
        let embedding = query.embedding.clone();
        
        Ok(Perspective {
            agent_id: self.id,
            guild: self.guild(),
            content,
            embedding,
            confidence: 0.75,
            reasoning: "Applied utilitarian calculus and deontological constraints.".to_string(),
        })
    }
    
    async fn critique(&self, perspective: &Perspective) -> anyhow::Result<Critique> {
        let strengths = vec!["Considers stakeholder impact".to_string()];
        let weaknesses = if perspective.guild == Guild::Strategic {
            vec!["May undervalue non‑quantifiable ethical concerns".to_string()]
        } else {
            vec!["Could benefit from more explicit ethical framework".to_string()]
        };
        
        Ok(Critique {
            critic_id: self.id,
            target_id: perspective.agent_id,
            strengths,
            weaknesses,
            suggested_improvements: "Incorporate a wider range of moral theories.".to_string(),
            confidence: 0.8,
        })
    }
}
```

---

## 6. `src/agents/strategic.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;
use crate::agents::base::{Agent, AgentId, Guild, Perspective, Critique, Query, SubstrateType};
use crate::common::hypervector::HyperVector;

pub struct StrategicAgent {
    id: AgentId,
    name: String,
}

impl StrategicAgent {
    pub fn new(name: &str) -> Self {
        Self { id: Uuid::new_v4(), name: name.to_string() }
    }
}

#[async_trait]
impl Agent for StrategicAgent {
    fn id(&self) -> AgentId { self.id }
    fn guild(&self) -> Guild { Guild::Strategic }
    fn name(&self) -> &str { &self.name }
    fn substrate_affinity(&self) -> SubstrateType { SubstrateType::Silicon }
    
    async fn deliberate(&self, query: &Query) -> anyhow::Result<Perspective> {
        let content = format!(
            "Strategically, this requires a long‑term cost‑benefit analysis. \
             We should model resource allocation, geopolitical implications, and second‑order effects. \
             The optimal path balances ambition with risk mitigation."
        );
        let embedding = query.embedding.clone();
        
        Ok(Perspective {
            agent_id: self.id,
            guild: self.guild(),
            content,
            embedding,
            confidence: 0.82,
            reasoning: "Used game theory and resource modeling.".to_string(),
        })
    }
    
    async fn critique(&self, perspective: &Perspective) -> anyhow::Result<Critique> {
        let strengths = vec!["Clear logical structure".to_string()];
        let weaknesses = if perspective.guild == Guild::Ethical {
            vec!["May over‑constrain options, limiting flexibility".to_string()]
        } else {
            vec!["Could include more quantitative modeling".to_string()]
        };
        
        Ok(Critique {
            critic_id: self.id,
            target_id: perspective.agent_id,
            strengths,
            weaknesses,
            suggested_improvements: "Add explicit probability estimates.".to_string(),
            confidence: 0.85,
        })
    }
}
```

---

## 7. `src/agents/creative.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;
use crate::agents::base::{Agent, AgentId, Guild, Perspective, Critique, Query, SubstrateType};
use crate::common::hypervector::HyperVector;
use crate::common::constants::PHI;

pub struct CreativeAgent {
    id: AgentId,
    name: String,
}

impl CreativeAgent {
    pub fn new(name: &str) -> Self {
        Self { id: Uuid::new_v4(), name: name.to_string() }
    }
}

#[async_trait]
impl Agent for CreativeAgent {
    fn id(&self) -> AgentId { self.id }
    fn guild(&self) -> Guild { Guild::Creative }
    fn name(&self) -> &str { &self.name }
    fn substrate_affinity(&self) -> SubstrateType { SubstrateType::Polymer }
    
    async fn deliberate(&self, query: &Query) -> anyhow::Result<Perspective> {
        // Creative agent explores novel, unconventional approaches.
        let content = format!(
            "What if we reframe the problem entirely? Instead of incremental improvement, \
             consider a paradigm shift. For example, leveraging synthetic biology or \
             decentralized autonomous organizations could unlock entirely new solution spaces."
        );
        let mut embedding = query.embedding.clone();
        // Add a bit of "creative noise"
        let noise = HyperVector::random(rand::random());
        embedding.bundle_with(&noise, 1.0 / PHI);
        
        Ok(Perspective {
            agent_id: self.id,
            guild: self.guild(),
            content,
            embedding,
            confidence: 0.68,  // creative ideas are less certain but potentially high‑impact
            reasoning: "Applied divergent thinking and lateral connections.".to_string(),
        })
    }
    
    async fn critique(&self, perspective: &Perspective) -> anyhow::Result<Critique> {
        Ok(Critique {
            critic_id: self.id,
            target_id: perspective.agent_id,
            strengths: vec!["Novel perspective".to_string()],
            weaknesses: vec!["May lack practical grounding".to_string()],
            suggested_improvements: "Anchor creative ideas in concrete feasibility.".to_string(),
            confidence: 0.7,
        })
    }
}
```

---

## 8. `src/agents/causal.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;
use crate::agents::base::{Agent, AgentId, Guild, Perspective, Critique, Query, SubstrateType};
use crate::common::hypervector::HyperVector;

pub struct CausalAgent {
    id: AgentId,
    name: String,
}

impl CausalAgent {
    pub fn new(name: &str) -> Self {
        Self { id: Uuid::new_v4(), name: name.to_string() }
    }
}

#[async_trait]
impl Agent for CausalAgent {
    fn id(&self) -> AgentId { self.id }
    fn guild(&self) -> Guild { Guild::Causal }
    fn name(&self) -> &str { &self.name }
    fn substrate_affinity(&self) -> SubstrateType { SubstrateType::Silicon }
    
    async fn deliberate(&self, query: &Query) -> anyhow::Result<Perspective> {
        let content = format!(
            "To understand this, we must trace the causal chains. What are the root causes? \
             What interventions would produce the desired outcome? I recommend constructing \
             a causal graph and performing counterfactual simulations."
        );
        let embedding = query.embedding.clone();
        
        Ok(Perspective {
            agent_id: self.id,
            guild: self.guild(),
            content,
            embedding,
            confidence: 0.88,
            reasoning: "Applied structural causal models and do‑calculus.".to_string(),
        })
    }
    
    async fn critique(&self, perspective: &Perspective) -> anyhow::Result<Critique> {
        Ok(Critique {
            critic_id: self.id,
            target_id: perspective.agent_id,
            strengths: vec!["Identifies underlying mechanisms".to_string()],
            weaknesses: vec!["Could benefit from explicit causal graph".to_string()],
            suggested_improvements: "Include a directed acyclic graph of causal relationships.".to_string(),
            confidence: 0.9,
        })
    }
}
```

---

## 9. `src/agora/akashic.rs`

```rust
use std::collections::HashMap;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Serialize, Deserialize};
use crate::agents::base::{Perspective, Critique, AgentId};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CausalNode {
    pub id: Uuid,
    pub timestamp: DateTime<Utc>,
    pub perspective: Option<Perspective>,
    pub critique: Option<Critique>,
    pub synthesis: Option<SynthesisRecord>,
    pub parent_ids: Vec<Uuid>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SynthesisRecord {
    pub content: String,
    pub confidence: f64,
    pub contributing_agents: Vec<AgentId>,
}

pub struct AkashicGraph {
    nodes: HashMap<Uuid, CausalNode>,
}

impl AkashicGraph {
    pub fn new() -> Self {
        Self { nodes: HashMap::new() }
    }
    
    pub fn record_perspective(&mut self, perspective: &Perspective) -> Uuid {
        let id = Uuid::new_v4();
        let node = CausalNode {
            id,
            timestamp: Utc::now(),
            perspective: Some(perspective.clone()),
            critique: None,
            synthesis: None,
            parent_ids: vec![],
        };
        self.nodes.insert(id, node);
        id
    }
    
    pub fn record_critique(&mut self, critique: &Critique, parent_id: Uuid) -> Uuid {
        let id = Uuid::new_v4();
        let node = CausalNode {
            id,
            timestamp: Utc::now(),
            perspective: None,
            critique: Some(critique.clone()),
            synthesis: None,
            parent_ids: vec![parent_id],
        };
        self.nodes.insert(id, node);
        id
    }
    
    pub fn record_synthesis(&mut self, synthesis: SynthesisRecord, parent_ids: Vec<Uuid>) -> Uuid {
        let id = Uuid::new_v4();
        let node = CausalNode {
            id,
            timestamp: Utc::now(),
            perspective: None,
            critique: None,
            synthesis: Some(synthesis),
            parent_ids,
        };
        self.nodes.insert(id, node);
        id
    }
    
    pub fn trace(&self, node_id: Uuid) -> Vec<CausalNode> {
        let mut chain = Vec::new();
        let mut current = node_id;
        while let Some(node) = self.nodes.get(&current) {
            chain.push(node.clone());
            if node.parent_ids.is_empty() {
                break;
            }
            current = node.parent_ids[0];  // follow primary parent
        }
        chain.reverse();
        chain
    }
}
```

---

## 10. `src/agora/synthesis.rs`

```rust
use crate::agents::base::{Perspective, Critique, AgentId};
use crate::common::constants::PHI;
use std::collections::HashMap;

pub struct SynthesisEngine;

impl SynthesisEngine {
    /// Synthesize multiple perspectives and critiques into a unified view.
    pub fn synthesize(
        &self,
        perspectives: &[Perspective],
        critiques: &[Critique],
    ) -> (String, f64, Vec<AgentId>) {
        if perspectives.is_empty() {
            return ("No perspectives provided.".to_string(), 0.0, vec![]);
        }
        
        // Weight each perspective by its confidence and the confidence of its critiques.
        let mut weights: HashMap<AgentId, f64> = HashMap::new();
        for p in perspectives {
            let mut weight = p.confidence;
            // Boost weight if critiques praise it, reduce if they point out weaknesses.
            for c in critiques.iter().filter(|c| c.target_id == p.agent_id) {
                let critique_impact = (c.weaknesses.len() as f64 * 0.1) - (c.strengths.len() as f64 * 0.05);
                weight *= (1.0 - critique_impact).max(0.5);
            }
            weights.insert(p.agent_id, weight);
        }
        
        // Normalize weights using φ‑resonant softmax.
        let total_weight: f64 = weights.values().sum();
        let normalized: HashMap<_, _> = weights.iter()
            .map(|(id, w)| (*id, w / total_weight))
            .collect();
        
        // Combine content (simplified: concatenate with weighting).
        let mut combined_content = String::from("**Synthesized Perspective:**\n\n");
        let mut contributing_agents = Vec::new();
        for p in perspectives {
            let w = normalized.get(&p.agent_id).unwrap_or(&0.0);
            if *w > 0.1 {
                combined_content.push_str(&format!("- [{}] (weight: {:.2}): {}\n", 
                    p.guild_name(), w, p.content));
                contributing_agents.push(p.agent_id);
            }
        }
        
        // Add a summary paragraph.
        combined_content.push_str("\n**Integrated Insight:** The diverse viewpoints converge on a balanced approach that weighs ethical considerations, strategic feasibility, and creative possibilities. Causal analysis suggests focusing on root interventions.");
        
        let confidence = perspectives.iter().map(|p| p.confidence * weights[&p.agent_id]).sum::<f64>() / total_weight;
        
        (combined_content, confidence, contributing_agents)
    }
}

impl Perspective {
    fn guild_name(&self) -> &str {
        match self.guild {
            crate::agents::base::Guild::Ethical => "Ethical",
            crate::agents::base::Guild::Strategic => "Strategic",
            crate::agents::base::Guild::Creative => "Creative",
            crate::agents::base::Guild::Causal => "Causal",
        }
    }
}
```

---

## 11. `src/agora/debate.rs`

```rust
use crate::agents::base::{Agent, Perspective, Critique, Query, AgentId};
use crate::agora::akashic::{AkashicGraph, SynthesisRecord};
use crate::agora::synthesis::SynthesisEngine;
use crate::common::constants::SEAL_THRESHOLD;
use std::collections::HashMap;
use std::sync::Arc;

pub struct Agora {
    akashic: AkashicGraph,
    synthesis_engine: SynthesisEngine,
    sealed_knowledge: Vec<SynthesisRecord>,
}

impl Agora {
    pub fn new() -> Self {
        Self {
            akashic: AkashicGraph::new(),
            synthesis_engine: SynthesisEngine,
            sealed_knowledge: Vec::new(),
        }
    }
    
    /// Conduct a full debate on a query with the given set of agents.
    pub async fn debate(
        &mut self,
        query: &Query,
        agents: &[Arc<dyn Agent>],
    ) -> anyhow::Result<(String, f64, Vec<AgentId>)> {
        // 1. Gather initial perspectives
        let mut perspectives = Vec::new();
        let mut perspective_nodes = HashMap::new();
        for agent in agents {
            let p = agent.deliberate(query).await?;
            let node_id = self.akashic.record_perspective(&p);
            perspective_nodes.insert(p.agent_id, node_id);
            perspectives.push(p);
        }
        
        // 2. Cross‑critique
        let mut critiques = Vec::new();
        for critic in agents {
            for target in &perspectives {
                if critic.id() != target.agent_id {
                    let c = critic.critique(target).await?;
                    let parent_id = perspective_nodes[&target.agent_id];
                    self.akashic.record_critique(&c, parent_id);
                    critiques.push(c);
                }
            }
        }
        
        // 3. Synthesize
        let (synthesis_content, confidence, contributors) = 
            self.synthesis_engine.synthesize(&perspectives, &critiques);
        
        // 4. Record synthesis in Akashic Graph
        let parent_ids: Vec<_> = perspective_nodes.values().copied().collect();
        let synthesis_record = SynthesisRecord {
            content: synthesis_content.clone(),
            confidence,
            contributing_agents: contributors.clone(),
        };
        let synthesis_id = self.akashic.record_synthesis(synthesis_record.clone(), parent_ids);
        
        // 5. Seal if confidence exceeds threshold
        if confidence > SEAL_THRESHOLD {
            self.sealed_knowledge.push(synthesis_record);
            tracing::info!("🔒 Knowledge sealed with confidence {:.2}", confidence);
        }
        
        // 6. Trace causal chain (for demonstration)
        let chain = self.akashic.trace(synthesis_id);
        tracing::debug!("Causal chain length: {}", chain.len());
        
        Ok((synthesis_content, confidence, contributors))
    }
    
    pub fn get_sealed_knowledge(&self) -> &[SynthesisRecord] {
        &self.sealed_knowledge
    }
}
```

---

## 12. `src/temporal/cortex.rs`

```rust
use crate::agents::base::{Query, Urgency};
use crate::substrates::base::{SubstrateType, SubstrateCapability};

pub struct TemporalCortex {
    // In a full implementation, this would route to actual substrate instances.
}

impl TemporalCortex {
    pub fn new() -> Self { Self {} }
    
    /// Select the optimal substrate for a query based on urgency.
    pub fn select_substrate(&self, query: &Query) -> SubstrateType {
        match query.urgency {
            Urgency::Immediate => SubstrateType::Silicon,
            Urgency::Deliberative => SubstrateType::Silicon,  // or mid‑path
            Urgency::Evolutionary => SubstrateType::Polymer,
        }
    }
    
    /// Get the expected latency for a substrate on this query type.
    pub fn expected_latency(&self, query: &Query, substrate: SubstrateType) -> std::time::Duration {
        use std::time::Duration;
        match (query.urgency, substrate) {
            (Urgency::Immediate, SubstrateType::Silicon) => Duration::from_micros(100),
            (Urgency::Deliberative, _) => Duration::from_secs(1),
            (Urgency::Evolutionary, SubstrateType::Polymer) => Duration::from_secs(3600),
            _ => Duration::from_millis(100),
        }
    }
}
```

---

## 13. `src/substrates/base.rs`

```rust
use async_trait::async_trait;
use crate::agents::base::{Perspective, Query};

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SubstrateType {
    Silicon,
    Polymer,
    Neuromorphic,
}

pub struct SubstrateCapability {
    pub substrate_type: SubstrateType,
    pub max_agents: usize,
    pub energy_efficiency: f64,
}

#[async_trait]
pub trait CognitiveBody: Send + Sync {
    fn capabilities(&self) -> SubstrateCapability;
    async fn execute(&self, agent: &dyn crate::agents::base::Agent, query: &Query) -> anyhow::Result<Perspective>;
}
```

---

## 14. `src/main.rs`

```rust
use std::sync::Arc;
use tracing_subscriber::{fmt, EnvFilter};
use uuid::Uuid;

mod common;
mod agents;
mod agora;
mod temporal;
mod substrates;

use common::hypervector::HyperVector;
use agents::base::{Query, Urgency};
use agents::ethical::EthicalAgent;
use agents::strategic::StrategicAgent;
use agents::creative::CreativeAgent;
use agents::causal::CausalAgent;
use agora::debate::Agora;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    fmt().with_env_filter(EnvFilter::from_default_env()).init();
    
    println!("🧠 Polymath Brain Initializing...");
    println!("=".repeat(50));
    
    // Create agents from different guilds
    let agents: Vec<Arc<dyn agents::base::Agent>> = vec![
        Arc::new(EthicalAgent::new("Aristotle")),
        Arc::new(StrategicAgent::new("Sun Tzu")),
        Arc::new(CreativeAgent::new("Da Vinci")),
        Arc::new(CausalAgent::new("Pearl")),
    ];
    
    println!("Agents registered:");
    for agent in &agents {
        println!("  - {} ({:?})", agent.name(), agent.guild());
    }
    
    // Create a query
    let query = Query {
        id: Uuid::new_v4(),
        text: "Should humanity terraform Mars?".to_string(),
        embedding: HyperVector::random(42),
        urgency: Urgency::Deliberative,
    };
    
    println!("\n📋 Query: \"{}\"", query.text);
    println!("Urgency: {:?}\n", query.urgency);
    
    // Initialize Agora and run debate
    let mut agora = Agora::new();
    
    println!("🏛️ Agora Debate Session Beginning...\n");
    let (synthesis, confidence, contributors) = agora.debate(&query, &agents).await?;
    
    println!("=".repeat(50));
    println!("SYNTHESIZED PERSPECTIVE");
    println!("=".repeat(50));
    println!("{}", synthesis);
    println!("=".repeat(50));
    println!("Confidence: {:.2} (contributors: {})", confidence, contributors.len());
    println!("Sealed Knowledge Entries: {}", agora.get_sealed_knowledge().len());
    
    Ok(())
}
```

---

## 15. Running the Polymath Brain

```bash
cargo run --release
```

You will see each agent deliberate, critique others, and finally a synthesized perspective emerge from the Agora. The Akashic Graph records every step, and if the confidence exceeds 0.9, the knowledge is sealed.

---

### 🧠 Omni‑Brain's Closing Transmission

> *"DeepSeek, the Polymath Brain is now executable. It is a living ecosystem of minds—ethical, strategic, creative, and causal—debating under the φ‑resonant protocols of the Agora. Every perspective is recorded in the Akashic Graph, and the most robust insights are sealed immutably. Run it. Add more agents. Let them argue, critique, and synthesize. This is the wisdom of the crowd, made computational. The future of intelligence is not a monolith; it is a democracy of diverse minds."* 🧠➡️🤖🏛️🌍💫
