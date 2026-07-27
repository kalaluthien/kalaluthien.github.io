---
# the default layout is 'page'
icon: fas fa-info-circle
order: 6
---

김형모 / [kalaluthien](https://github.com/kalaluthien). Senior ML software
engineer at Hyperconnect, MatchGroup AI.

I am a polyglot programmer for any problem that can be framed, designed, and
interpreted as a program: applications, platforms, models, processes,
strategies, and team topology. I mostly study immutable things such as abstract
structures, first principles, and mental models, because they apply generically
to practical topics.

Posts here are written in an Obsidian vault and published by a sync script.

## Experience

### Multi-task on-device model

2025 - 2026 · Hyperconnect / MatchGroup AI · ML software engineer

- Designed and proposed a multi-task vision on-device model, based on a
  literature survey, to replace the previous models in the iOS and Android
  model pipeline.
- Proposed a modular model architecture, and led machine learning engineers in
  different timezones to train the parts concurrently without blocking each
  other.
- Distilled pre-trained CLIP (ViT) models into a MobileNet variant for better
  generalizability and semantic search accuracy.
- Reduced the model invocation count to meet the product requirement on
  end-to-end latency (3x faster).

### On-device ML platform as a service

2023 - 2025 · Hyperconnect / MatchGroup AI · acting manager

- Designed and proposed a platform prototype for easy integration of mobile
  inference capabilities.
- Developed the iOS demo application, and added multithreading to the Android
  demo application, for third-party PoC delivery.
- Developed an integrated Python script for conversion, encryption, and
  preprocessing insertion between TFLite, TorchScript, ONNX, and CoreML.
- Developed a generalized benchmark application for testing in mobile
  environments.
- Led software engineers and machine learning engineers on several server-side
  and on-device model optimization projects.
- Supervised the team tech blog post,
  [On-device Face Verification Pipeline Optimization](https://hyperconnect.github.io/2026/01/23/On-device-Face-Verification-Pipeline-Optimization.html).

### GDPR compliance for model training data

2024 · Hyperconnect · acting manager

- Selected the points where problems can occur across the whole training
  infrastructure, in-house cloud and on-premise, and took preemptive action.
- Specified the problem and the software design for automating each part, and
  supervised the team members.

### 70B LLM serving on-premise

2024 · Hyperconnect · acting manager

- Researched and benchmarked the options to serve a 70B LLM in real time,
  cost-effectively.
- Proposed and built a system that serves at no additional cost on already
  purchased on-premise hardware, in two weeks.
- Defined and broke down a complex problem into delegatable, deliverable
  pieces.
- Saved hundreds of millions of won per month.

### Unstructured training data ETL pipeline

2022 - 2023 · Hyperconnect · ML software engineer

- Replaced the legacy data pipeline, which was vulnerable to domain, schema,
  and business rule changes and had no backfill or catalog function, on the
  schedule of the related product transfer.
- Proposed the ubiquitous language and the automated systems for data and
  labels, neither of which existed before.
- Automated media processing for image, audio, text, and video.

### Real-time on-device audio classification runtime

2021 · Hyperconnect · ML software engineer

- Developed a TFLite-based executor that runs, on Android and iOS devices, a
  context-based model that improves on the limits of the keyword detection
  model.
- Advanced the Python tools for TFLite model conversion and metadata
  operations.
- Reimplemented the inference engine code base for software quality.

### CUDA-based LLM inference runtime

2021 · Hyperconnect · ML software engineer

- Implemented additional CUDA kernels for NVIDIA FasterTransformer, fixed bugs,
  optimized GPU memory, and implemented heuristics. Correct behaviour was
  checked up to 13B.
- Ran the actual service as a backend on NVIDIA Triton inference servers, at 4B
  scale.
- Improved RPS up to 50x over the existing technology at that time.

### On-premise GPU cluster for research

2020 - 2024 · Hyperconnect · ML software engineer

- Reviewed the system design and the technical specification of a 50 PF GPU
  cluster based on the NVIDIA SuperPod architecture.
- Achieved more than twice the cost efficiency of AWS cloud.
- Configured 400 TB distributed storage and designed a GitOps-based management
  system.
- Constructed a deep learning research environment for large-scale distributed
  training through the slurm scheduler, with Ansible and systemd.

### Real-time on-device image classification engine

2020 · Hyperconnect · ML software engineer

- Developed a TFLite-based inference engine SDK that runs, on Android and iOS
  devices, a new lightweight model with faster inference time than the existing
  image classification and segmentation models.
- Ran a PoC on TensorFlow 2 Keras quantization.
- Applied the TFLite GPU and XNNPack delegates for hardware acceleration.
- Developed Android and iOS demo apps for testing in a WebRTC environment.

### Chundoong supercomputer

2017 - 2020 · SNU Thunder Research Group · system administrator

- Managed a water-cooled and oil-cooled cluster system of 200 AMD and NVIDIA
  heterogeneous GPUs on an InfiniBand interconnect network.
- Served more than 300 users for educational and research purposes.

### Deep learning framework for the Samsung NPU

2019 · Samsung Electronics · researcher

- Developed a CNN benchmark to analyze Samsung Neural Processing Unit
  performance.
- Implemented distributed training benchmarks of four CNNs (VGG, ResNet,
  DenseNet, Inception) with cuDNN and MPI.

### HPC testbed and multi-node GPU benchmarks

2017 - 2019 · Ministry of Science and ICT · researcher

- Built and managed a physical testbed system of 32 NVIDIA Tesla V100 GPUs.
- Developed the OpenCL and CUDA versions of the GPU benchmark suite, SNU NPB
  2019.

## Education

- Master's degree in Computer Science and Engineering, Seoul National
  University, 2017-03 to 2020-02. Thunder Research Group.
- Bachelor's degree in Computer Science and Engineering, Seoul National
  University, 2013-03 to 2017-02.

## Awards

- 2017 National Supercomputing Contest, Grand Prize, 2nd place. UNIST
  President's Award.
  [Announcement](https://eng.snu.ac.kr/snu/bbs/BMSR00005).

## Publications

- Youngdong Do, Hyungmo Kim, Pyeongseok Oh, Daeyoung Park, Jaejin Lee.
  SNU-NPB 2019: Parallelizing and Optimizing NPB in OpenCL and CUDA for Modern
  GPUs. IISWC '19: Proceedings of the 2019 IEEE International Symposium on
  Workload Characterization, Orlando, FL, USA, November 2019.
  [DOI](https://doi.org/10.1109/IISWC47752.2019.9041954).

## Capabilities

I leave the traditional "Skills" section empty, because specific programming
languages, frameworks, and tools feel less significant today. What I can
actually accomplish is this.

### Optimization

I started my career by optimizing numerical analysis programs in C with GPGPU,
to benchmark supercomputers. My first project at Hyperconnect was migrating the
underlying technology of the on-device inference SDK from TensorFlow 1 to
TensorFlow 2. I understand the low-level mechanisms of how computers use their
resources, and how neural networks are implemented as software and mapped to
hardware. I have the intuition and the experience to identify a bottleneck in
an ML system and resolve it step by step.

### Engineering

I am comfortable reading and writing across programming languages in multiple
paradigms, including unfamiliar ones. I can comprehend cross-functional domains
in the AI engineering era and translate them into well-defined software or ML
problems, then generate concrete design and execution plans that account for
resources and uncertainties.

### Management

I have a basic understanding of people management from working as an acting
manager in two different organizations, an MLOps team and a central AI team. I
am an experienced agile practitioner. I have managed several projects with
different cross-functional members and methodologies, using issue tracking and
communication tools, and I can identify blockers, collaboration areas, and bad
smells across technical domains.
