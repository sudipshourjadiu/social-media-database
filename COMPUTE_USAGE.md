# **Compute and Memory Requirements for SQL vs. NoSQL in a Social Network Feed**  

To better understand the infrastructure demands of generating a social network feed, let’s estimate the required **CPU (GHz) and RAM (GB)** for both **SQL (normalized) and NoSQL (denormalized)** approaches.  

## **Assumptions & Baseline Metrics**  
- **10,000 daily active users (DAUs)**  
- **60 feed requests per user per hour** (10 posts per minute)  
- **Peak traffic**: ~166 QPS (queries per second)  
- **Avg. response time target**: <100ms  
- **SQL**: Uses **B-tree indexes**, ~130 lookups per feed request  
- **NoSQL**: Uses **denormalized documents**, 1 read per feed request  

---

## **1. SQL Database (Normalized) Requirements**  
### **Query Workload**  
- **~21,667 index lookups/sec** (166 QPS × 130 lookups)  
- **Joins required**: Posts + Users + Images + Comments + Reactions  

### **CPU Estimation**  
- Each **B-tree lookup** takes ~0.1ms (optimistic, assuming cached index).  
- **Total CPU time per second**:  
  - `21,667 lookups × 0.1ms = 2,166.7ms (≈2.17 CPU-seconds per second)`  
  - **≈2.17 CPU cores at 100% utilization** (just for lookups, excluding joins and I/O).  
- **Real-world overhead**:  
  - Joins add **~5–10x more CPU load** (parsing rows, merging data).  
  - **Estimated required CPU**: **10–20 vCPU cores** (modern cloud CPUs run at ~3 GHz).  

### **RAM Estimation**  
- **Indexes must fit in RAM** for fast lookups.  
- Estimated **index size**:  
  - **Posts table**: ~30,000 posts × 500B ≈ **15 MB**  
  - **Users table**: ~10,000 users × 1KB ≈ **10 MB**  
  - **Comments table**: ~150,000 comments (5 per post) × 500B ≈ **75 MB**  
  - **Total indexes**: ~**100–200 MB** (compressed)  
- **Working set (hot data)**:  
  - Recent posts + active users should stay in memory.  
  - **Recommended RAM**: **8–16 GB** (to cache indexes + frequent queries).  

### **SQL Server Recommendation**  
- **CPU**: **16 vCPU cores (~3 GHz each)**  
- **RAM**: **16 GB+**  
- **Storage**: **Fast SSD (NVMe) for low-latency disk access**  

---

## **2. NoSQL (MongoDB - Denormalized) Requirements**  
### **Query Workload**  
- **166 document reads/sec** (1 per feed request)  
- **No joins** (all data embedded in a single document).  

### **CPU Estimation**  
- Each **document read** takes ~0.5ms (optimized, indexed `_id`).  
- **Total CPU time per second**:  
  - `166 reads × 0.5ms = 83ms (≈0.083 CPU-seconds per second)`  
  - **<<1 CPU core needed** (for reads alone).  
- **Real-world overhead**:  
  - Deserialization, aggregation, and network I/O add some load.  
  - **Estimated required CPU**: **2–4 vCPU cores** (at ~3 GHz).  

### **RAM Estimation**  
- **Working set (hot data)**:  
  - Recent posts (~30,000) × 10KB (denormalized doc) ≈ **300 MB**  
  - **Indexes**: `timestamp` and `user_id` ≈ **50 MB**  
- **Recommended RAM**: **4–8 GB** (enough to cache active posts).  

### **MongoDB Server Recommendation**  
- **CPU**: **4 vCPU cores (~3 GHz each)**  
- **RAM**: **8 GB**  
- **Storage**: **SSD (but less critical than SQL due to fewer disk seeks)**  

---

## **3. Key Takeaways: SQL vs. NoSQL Resource Comparison**  
| **Metric**               | **SQL (Normalized)**       | **NoSQL (Denormalized)** |  
|--------------------------|---------------------------|--------------------------|  
| **CPU Required**         | **16+ vCPU cores**        | **4 vCPU cores**         |  
| **RAM Required**         | **16 GB+**                | **8 GB**                 |  
| **Disk I/O Pressure**    | **High (many seeks)**     | **Low (sequential reads)** |  
| **Scaling Complexity**   | **Hard (joins bottleneck)** | **Easy (sharding)**      |  

### **Why NoSQL (MongoDB) Wins for Feed Generation**  
1. **10x Lower CPU Usage** (4 cores vs. 16+ cores).  
2. **2x Less RAM Needed** (8 GB vs. 16 GB).  
3. **No Joins = Faster Response Times** (consistently <50ms).  
4. **Easier Horizontal Scaling** (sharding by `user_id` or `timestamp`).  

---

## **Final Infrastructure Recommendations**  
### **For SQL (If You Must Use It)**  
- **Database**: PostgreSQL (with aggressive caching).  
- **CPU**: 16+ vCPU cores.  
- **RAM**: 16–32 GB (to cache indexes + working set).  
- **Disk**: NVMe SSDs (to reduce seek latency).  

### **For NoSQL (Optimal Choice)**  
- **Database**: MongoDB (with embedded documents).  
- **CPU**: 4–8 vCPU cores.  
- **RAM**: 8–16 GB (for caching active posts).  
- **Disk**: Standard SSD (I/O is less critical).  

### **Additional Optimizations**  
- **Redis Caching**: Store hot feeds in memory (~1 GB RAM cache).  
- **Read Replicas**: Distribute feed load.  
- **CDN for Images**: Offload image serving.  

---

## **Conclusion**  
For a social network feed at **10,000 DAUs (166 QPS)**, **NoSQL (MongoDB) reduces CPU needs by 75% and RAM by 50% compared to SQL**, while also simplifying scaling. **Denormalization is the clear winner** for high-throughput, low-latency feed generation.  
