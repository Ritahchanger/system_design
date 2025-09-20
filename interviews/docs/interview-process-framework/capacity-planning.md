# Capacity Planning Interview Guide
## System Design & Business Operations Focus

---

## 📚 RELATED DOCUMENTATION

### Engineer Interview Guides
- **[Junior Engineer Guide](../engineer-interview-guide/junior-guide.md)** - Entry-level technical interviews
- **[Senior Engineer Guide](../engineer-interview-guide/senior-guide.md)** - Advanced technical leadership interviews
- **[Interview Process](../engineer-interview-guide/interview-process.md)** - Complete interview workflow and procedures

### Estimation Techniques
- **[Trade-offs & Decisions](../estimation-techniques/trade-offs-decisions.md)** - Business estimation and decision-making frameworks

### Practice Resources
- **[Practice Questions](../practise-questions/)** - Hands-on interview scenarios and solutions

### Quick Navigation
```
📁 system_design/interviews/docs/
├── 🔧 engineer-interview-guide/
│   ├── interview-process.md
│   ├── junior-guide.md
│   └── senior-guide.md
├── 📊 estimation-techniques/
│   └── trade-offs-decisions.md
├── ⚡ interview-process-framework/
│   └── capacity-planning.md (current)
└── 💡 practise-questions/
```

---

## CROSS-REFERENCE SCENARIOS

### For Junior Engineers
> **See:** [Junior Guide](../engineer-interview-guide/junior-guide.md) for simplified capacity planning questions
- Basic load calculations
- Single-server scaling decisions
- Resource monitoring fundamentals

### For Senior Engineers  
> **See:** [Senior Guide](../engineer-interview-guide/senior-guide.md) for advanced system architecture
- Multi-region capacity distribution
- Complex auto-scaling strategies
- Performance optimization at scale

### Business Context Integration
> **See:** [Trade-offs & Decisions](../estimation-techniques/trade-offs-decisions.md) for business estimation
- ROI calculations for infrastructure investments
- Market sizing for capacity requirements
- Cost-benefit analysis frameworks

---

---

## OVERVIEW

This document provides a comprehensive framework for evaluating candidates' capacity planning skills across both technical systems and business operations. Questions range from infrastructure scaling to resource allocation and performance optimization.

### 🔗 Integration with Other Guides
This guide works in conjunction with our complete interview framework:
- **Technical Foundation**: Build on concepts from [Junior](../engineer-interview-guide/junior-guide.md) and [Senior](../engineer-interview-guide/senior-guide.md) engineering guides
- **Business Context**: Apply estimation techniques from [Trade-offs & Decisions](../estimation-techniques/trade-offs-decisions.md)
- **Process Framework**: Follow procedures outlined in [Interview Process](../engineer-interview-guide/interview-process.md)

---

## SECTION 1: TECHNICAL CAPACITY PLANNING

### Infrastructure Scaling

**Question 1: Web Server Capacity**
> "Design capacity planning for a web application expecting to grow from 10,000 to 1 million daily active users over 12 months."

**Expected Analysis Framework:**
```
Current State:
- 10,000 DAU
- Peak concurrent users: 2,000 (20% of DAU)
- Average session duration: 15 minutes
- Requests per session: 50
- Peak RPS: 2,000 × 50 ÷ (15 × 60) = 111 RPS

Target State (1M DAU):
- 1,000,000 DAU
- Peak concurrent users: 200,000
- Peak RPS: 11,111 RPS (100x increase)
```

**Scaling Strategy:**
- **Vertical scaling limits**: Single server max ~1,000 RPS
- **Horizontal scaling**: Need ~12-15 servers with load balancing
- **Database scaling**: Read replicas, sharding strategy
- **CDN implementation**: Static content delivery
- **Caching layers**: Redis/Memcached for hot data

**Follow-up Questions:**
- How would you handle traffic spikes during peak events?
- What monitoring metrics would you track?
- How would you test this capacity before going live?
- **See [Senior Guide](../engineer-interview-guide/senior-guide.md)** for advanced scaling patterns

---

**Question 2: Database Capacity Planning**
> "A MySQL database currently handles 1,000 QPS with 100GB of data. Plan for scaling to 10,000 QPS and 1TB of data."

**Analysis Components:**

**Read/Write Ratio Analysis:**
```
Current: 1,000 QPS (assume 80% reads, 20% writes)
- Read QPS: 800
- Write QPS: 200

Target: 10,000 QPS
- Read QPS: 8,000
- Write QPS: 2,000
```

**Scaling Strategy:**
```
Read Scaling:
- Master + 4-6 read replicas
- Connection pooling
- Query optimization
- Read caching layer

Write Scaling:
- Vertical scaling of master
- Write-through caching
- Asynchronous processing
- Database sharding (if needed)

Storage Scaling:
- SSD storage for performance
- Automated backups
- Archiving strategy for old data
```

**Capacity Timeline:**
- **Month 1-3**: Add read replicas
- **Month 4-6**: Implement caching
- **Month 7-9**: Optimize queries, add monitoring
- **Month 10-12**: Consider sharding if needed

---

### Cloud Infrastructure

**Question 3: AWS Auto-Scaling Design**
> "Design an auto-scaling strategy for an e-commerce platform with predictable daily patterns and unpredictable promotional spikes."

**Scaling Metrics to Consider:**
- CPU utilization (target: 70%)
- Memory utilization (target: 80%)
- Request latency (target: <200ms)
- Queue depth for background jobs

**Auto-scaling Configuration:**
```
Predictable Scaling (Schedule-based):
- Morning ramp-up: 6 AM - 10 AM (+50% capacity)
- Peak hours: 10 AM - 8 PM (100% capacity)
- Evening scale-down: 8 PM - 12 AM (-30% capacity)
- Night minimum: 12 AM - 6 AM (20% capacity)

Dynamic Scaling (Metric-based):
- Scale-out trigger: CPU > 70% for 2 minutes
- Scale-in trigger: CPU < 30% for 5 minutes
- Maximum instances: 50
- Minimum instances: 5
```

**Promotional Spike Strategy:**
- **Pre-emptive scaling**: Manual scale-up before announcements
- **Circuit breakers**: Protect downstream services
- **Queue-based processing**: Handle traffic bursts
- **CDN optimization**: Reduce origin server load

> 💡 **Related Reading**: See [Trade-offs & Decisions](../estimation-techniques/trade-offs-decisions.md) for cost-performance trade-off analysis

---

### Performance Optimization

**Question 4: API Rate Limiting Design**
> "Design a rate limiting system for an API that serves 1 million requests per day across 10,000 different clients."

**Capacity Analysis:**
```
Average RPS: 1,000,000 ÷ (24 × 3600) = 11.6 RPS
Peak multiplier: 3x = 35 RPS
Per-client average: 100 requests/day
Per-client peak: ~1 RPS per client
```

**Rate Limiting Strategy:**
```
Global Limits:
- System-wide: 100 RPS
- Per-IP: 10 RPS
- Per-API-key: 5 RPS

Time Window Strategies:
- Fixed window: Simple, but allows bursts
- Sliding window: Smooth, more complex
- Token bucket: Allows controlled bursts

Implementation:
- Redis for distributed rate limiting
- Lua scripts for atomic operations
- Response headers for limit communication
```

---

## SECTION 2: BUSINESS CAPACITY PLANNING

### Staffing and Operations

**Question 5: Customer Support Capacity**
> "Plan staffing for a customer support team handling 1,000 tickets daily with target response time of 2 hours."

**Capacity Calculation Framework:**
```
Daily ticket volume: 1,000
Average handling time: 15 minutes
Target response time: 2 hours
Operating hours: 12 hours (7 AM - 7 PM)

Agent capacity per day: 12 hours × 60 min ÷ 15 min = 48 tickets
Required agents: 1,000 ÷ 48 = 21 agents

Response time consideration:
- Tickets arrive throughout 12 hours
- Average queue time should be < 2 hours
- Peak hour adjustment: +30% staffing
```

**Staffing Schedule:**
- **Morning shift (7-3)**: 8 agents
- **Day shift (11-7)**: 8 agents  
- **Peak overlap (11-3)**: 5 additional agents
- **Total**: 21 agents with shift overlaps

**Scaling Considerations:**
- **Seasonal variations**: Holiday season +40%
- **Channel optimization**: Self-service portal
- **Skill-based routing**: Complex issues to senior agents
- **Outsourcing strategy**: Overflow capacity

---

**Question 6: Warehouse Capacity Planning**
> "Design warehouse capacity for an e-commerce business growing from 1,000 to 10,000 orders per day."

**Current State Analysis:**
```
Orders per day: 1,000
Average items per order: 2.5
Daily item picks: 2,500
Pick rate per person: 100 items/hour
Daily hours of operation: 10 hours
Required pickers: 2,500 ÷ (100 × 10) = 2.5 → 3 people
```

**Future State Planning:**
```
Target orders: 10,000 daily
Daily item picks: 25,000
Required pickers: 25,000 ÷ (100 × 10) = 25 people

Additional capacity factors:
- Receiving/stocking: 5 people
- Packing/shipping: 8 people
- Quality control: 3 people
- Management/admin: 4 people
Total warehouse staff: 45 people
```

**Infrastructure Requirements:**
- **Storage space**: 50,000 sq ft (current: 10,000)
- **Picking zones**: 10 zones vs. current 2
- **Packing stations**: 15 stations vs. current 3
- **Technology**: WMS, barcode scanners, conveyors

> 🔗 **Cross-Reference**: For business estimation techniques, see [Trade-offs & Decisions Guide](../estimation-techniques/trade-offs-decisions.md#business-estimation--sizing)

---

### Retail Operations

**Question 7: Store Capacity Planning**
> "Plan optimal capacity for a electronics retail store expecting 200 customers per day with average visit duration of 20 minutes."

**Customer Flow Analysis:**
```
Daily customers: 200
Average visit duration: 20 minutes
Operating hours: 12 hours (8 AM - 8 PM)
Peak hours: 4 hours (typically 2-6 PM)

Average concurrent customers:
- Overall: (200 × 20 min) ÷ (12 × 60 min) = 5.6 customers
- Peak periods: ~15 customers (3x average)
```

**Space and Staff Planning:**
```
Store Layout:
- Customer area: 400 sq ft (25 sq ft per peak customer)
- Product display: 300 sq ft
- Storage/office: 200 sq ft
- Total: 900 sq ft minimum

Staffing Requirements:
- Sales staff: 2 people (peak), 1 (off-peak)
- Cashier: 1 person dedicated
- Manager/tech support: 1 person
- Peak staff: 4 people, Off-peak: 3 people
```

**Service Capacity:**
- **Sales consultation**: 5 customers/hour per staff
- **Checkout process**: 12 customers/hour per cashier
- **Technical support**: 2 customers/hour per tech staff

---

## SECTION 3: SYSTEM PERFORMANCE ANALYSIS

### Load Testing and Benchmarking

**Question 8: Load Testing Strategy**
> "Design a load testing plan for an API that needs to handle Black Friday traffic (expected 10x normal load)."

**Testing Phases:**

**Phase 1: Baseline Testing**
```
Normal load: 100 RPS
Duration: 30 minutes
Metrics: Response time, error rate, resource utilization
Goal: Establish performance baseline
```

**Phase 2: Stress Testing**
```
Incremental load: 100 → 1000 RPS over 15 minutes
Duration: 45 minutes total
Goal: Find breaking point and bottlenecks
```

**Phase 3: Spike Testing**
```
Sudden load: 100 → 1000 RPS instantly
Duration: 10 minutes at peak
Goal: Test auto-scaling and recovery
```

**Phase 4: Endurance Testing**
```
Sustained load: 500 RPS
Duration: 4 hours
Goal: Identify memory leaks, resource degradation
```

**Key Metrics to Monitor:**
- **Response time percentiles**: p50, p95, p99
- **Error rates**: 4xx, 5xx responses
- **Throughput**: Requests per second
- **Resource utilization**: CPU, memory, I/O
- **Database performance**: Query time, connections

> 📋 **Process Note**: Follow load testing procedures outlined in [Interview Process Guide](../engineer-interview-guide/interview-process.md)

---

### Monitoring and Alerting

**Question 9: Capacity Monitoring System**
> "Design a monitoring system to predict when you'll need to scale your infrastructure before hitting capacity limits."

**Monitoring Framework:**

**Leading Indicators:**
```
Growth Metrics:
- Daily/weekly active users growth rate
- Request volume trend (7-day, 30-day)
- Database growth rate
- Storage utilization trend

Performance Metrics:
- Response time trends (not just current values)
- Error rate increases
- Queue depth growth
- Resource utilization trends
```

**Predictive Alerting:**
```
CPU Utilization:
- Warning: Trending to hit 80% within 48 hours
- Critical: Current > 70% for 10+ minutes

Database Connections:
- Warning: Using > 70% of max connections
- Critical: Growth trend suggests 90% within 24 hours

Storage:
- Warning: Growth rate suggests 90% full within 30 days
- Critical: > 85% current utilization
```

**Capacity Planning Dashboard:**
- **Current vs. projected capacity**
- **Growth rate trending**
- **Resource utilization heatmaps**
- **Cost projection based on scaling needs**

---

## SECTION 4: ADVANCED SCENARIOS

### Multi-Region Capacity

**Question 10: Global Load Distribution**
> "Design capacity planning for a global application serving users in US, Europe, and Asia with 24/7 availability requirements."

**Regional Load Distribution:**
```
Total daily users: 1,000,000
Regional breakdown:
- US: 400,000 (40%)
- Europe: 350,000 (35%) 
- Asia: 250,000 (25%)

Time zone considerations:
- Peak hours shift by region
- US peak: 2-6 PM EST
- EU peak: 2-6 PM CET  
- Asia peak: 7-11 PM JST
```

**Capacity Strategy:**
```
Active-Active Multi-Region:
- US East: 40% capacity + 20% buffer
- EU West: 35% capacity + 20% buffer
- Asia Pacific: 25% capacity + 20% buffer

Failover Capacity:
- Each region can handle +50% load from adjacent regions
- Cross-region data replication with <1 second lag
- Auto-failover triggers and procedures
```

**Infrastructure Planning:**
- **CDN edge locations**: 50+ global points of presence
- **Database replication**: Master-master with conflict resolution
- **Load balancing**: GeoDNS routing with health checks
- **Monitoring**: Global dashboard with regional breakdowns

---

### Disaster Recovery Capacity

**Question 11: DR Capacity Planning**
> "Plan disaster recovery capacity for a financial services application with 99.9% availability requirement and 1-hour RTO."

**Availability Analysis:**
```
99.9% availability = 8.77 hours downtime/year
RTO: 1 hour (system must recover within 1 hour)
RPO: 15 minutes (max acceptable data loss)

Capacity requirements for DR:
- Hot standby: 100% capacity, immediate failover
- Warm standby: 70% capacity, 15-minute startup
- Cold standby: 0% capacity, 1+ hour startup

Financial services requirement: Hot standby mandatory
```

**DR Infrastructure:**
```
Primary Site:
- Active production: 100% capacity
- Real-time monitoring and health checks

DR Site:
- Hot standby: 100% capacity ready
- Real-time data replication
- Automated failover procedures
- Regular failover testing (monthly)

Network:
- Dedicated lines between sites
- Multiple ISP connections
- Health check endpoints every 30 seconds
```

---

## SECTION 5: COST OPTIMIZATION

### Resource Optimization

**Question 12: Cloud Cost vs. Performance**
> "Optimize cloud infrastructure costs while maintaining performance for a workload with predictable daily patterns."

**Cost Analysis Framework:**
```
Current setup: 10 × c5.large instances (24/7)
Monthly cost: 10 × $73 = $730

Usage pattern analysis:
- Peak hours (8 AM - 8 PM): Need 10 instances  
- Off-peak hours (8 PM - 8 AM): Need 3 instances
- Weekend: 50% of weekday traffic

Optimization strategies:
1. Auto-scaling: Scale down during off-peak
2. Reserved instances: 3 instances baseline
3. Spot instances: Burst capacity
4. Different instance types: Memory vs. CPU optimized
```

**Optimized Architecture:**
```
Reserved Instances (1-year): 3 × c5.large = $132/month
Auto-scaling On-demand: 0-7 additional instances
Spot instances: Burst capacity up to 15 instances

Estimated savings:
- Current: $730/month
- Optimized: $350-450/month
- Savings: 40-50% cost reduction
```

---

## SECTION 6: EVALUATION FRAMEWORK

### Assessment Criteria

**Technical Competency:**
- **Scalability thinking**: Considers both vertical and horizontal scaling
- **Bottleneck identification**: Recognizes system constraints
- **Monitoring approach**: Proactive vs. reactive strategies
- **Cost awareness**: Balances performance with budget

**Business Acumen:**
- **Growth planning**: Anticipates business needs
- **Risk management**: Considers failure scenarios
- **Resource allocation**: Optimizes human and technical resources
- **Stakeholder communication**: Explains technical concepts clearly

### Scoring Rubric

| **Criteria** | **Excellent (4)** | **Good (3)** | **Average (2)** | **Poor (1)** |
|-------------|------------------|--------------|----------------|--------------|
| **Problem Analysis** | Identifies all key factors | Covers main issues | Some analysis | Superficial |
| **Scaling Strategy** | Multiple approaches considered | Solid approach | Basic scaling | No clear strategy |
| **Calculations** | Accurate and justified | Mostly correct | Some errors | Major mistakes |
| **Monitoring** | Comprehensive monitoring plan | Good metrics identified | Basic monitoring | No monitoring plan |
| **Cost Consideration** | Balances cost and performance | Considers costs | Mentions cost | Ignores costs |
| **Communication** | Clear, structured explanation | Generally clear | Somewhat unclear | Confusing |

### Red Flags
- **Over-engineering**: Excessive complexity for the problem size
- **Under-planning**: No consideration of growth or failures
- **Cost ignorance**: No awareness of financial implications
- **Monitoring blindness**: No proactive monitoring strategy
- **Single point of failure**: Not considering redundancy

---

## INTERVIEWER GUIDELINES

### Question Progression
1. **Start simple**: Basic capacity calculations
2. **Add complexity**: Multiple variables and constraints
3. **Introduce trade-offs**: Cost vs. performance decisions
4. **Test adaptability**: Change requirements mid-discussion

### Probing Techniques
- "How would you validate this estimate?"
- "What happens if traffic is 3x higher than expected?"
- "How would you monitor this in production?"
- "What are the cost implications of this approach?"
- "How would you test this capacity plan?"

### Best Practices
- **Focus on thought process** over exact numbers
- **Encourage assumptions** to be stated clearly
- **Ask for alternatives** when they present a solution
- **Challenge edge cases** and failure scenarios
- **Relate to real-world experience** when possible

> 📖 **Additional Resources**:
> - **Junior candidates**: Use simplified examples from [Junior Guide](../engineer-interview-guide/junior-guide.md)
> - **Senior candidates**: Reference complex scenarios in [Senior Guide](../engineer-interview-guide/senior-guide.md)
> - **Business context**: Apply frameworks from [Trade-offs & Decisions](../estimation-techniques/trade-offs-decisions.md)

---

## CONCLUSION

This capacity planning interview guide assesses candidates' ability to:
- **Plan for scale** proactively
- **Balance competing priorities** (cost, performance, reliability)
- **Think systematically** about complex problems
- **Communicate technical concepts** clearly
- **Consider operational realities** beyond theoretical solutions

The best candidates will demonstrate structured thinking, practical experience, and the ability to adapt their solutions based on changing requirements and constraints.

---

## 🔗 NEXT STEPS

### For Interviewers:
1. **Review role requirements** with appropriate guide:
   - [Junior Engineers](../engineer-interview-guide/junior-guide.md) - Focus on basic capacity concepts
   - [Senior Engineers](../engineer-interview-guide/senior-guide.md) - Emphasize architecture and leadership
2. **Integrate business context** using [Trade-offs & Decisions](../estimation-techniques/trade-offs-decisions.md)
3. **Follow structured process** per [Interview Process](../engineer-interview-guide/interview-process.md)

### For Practice:
- **Hands-on scenarios**: Check [Practice Questions](../practise-questions/) directory
- **Mock interviews**: Use cross-referenced examples from other guides
- **Real-world application**: Apply frameworks to current projects

