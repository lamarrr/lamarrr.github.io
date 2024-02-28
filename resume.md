# Basit Ayantunde

**Location**: Manchester, United Kingdom
**Email**: [basitayde@gmail.com](mailto:basitayde@gmail.com)
**Phone**: +447533421146
**Github**: [github.com/lamarrr](github.com/lamarrr)
**LinkedIn**: [linkedin.com/in/basit-ayantunde](linkedin.com/in/basit-ayantunde)
**Website**: [basit.pro](https://basit.pro)
**Skills**: C++, C#, C, Java, Python, CMake, Vulkan, Unreal Engine, Linux, Windows

# EDUCATION

## University of Ilorin, Kwara, Nigeria, September 2016 – September 2021

B. Eng. Mechanical Engineering, Second Class Honors (Upper Division, 3.5/5 GPA)

# WORK EXPERIENCE

## Codethink - Manchester, United Kingdom

### Software Engineer, October 2023 - Present

- Implemented keyboard and touch input simulation APIs using Linux's _uinput_ and _evdev_ subsystems for our OSS tool: [QAD](https://www.codethink.co.uk/articles/2023/qad-for-hardware-testing/) used in automated remote testing of embedded GUIs [PR #36](https://gitlab.com/CodethinkLabs/qad/qad/-/merge_requests/36) (C, Embedded Linux)
- Improved [QAD's](https://gitlab.com/CodethinkLabs/qad/qad) screen capture latency by 26% by migrating the ILM screen capture backend to an async interface [PR #38](https://gitlab.com/CodethinkLabs/qad/qad/-/merge_requests/38) (C, Embedded Linux)
- Implemented a Conda license aggregator in ClearlyDefined for integration into our client's automated license/compliance testing pipeline [PR #532](https://github.com/clearlydefined/crawler/pull/532) (Javascript, Node.JS, React, SPDX)
- Implementing a Graph-Based Signal Subsystem template generator for our client's Advanced Driver Assistance System's models, expected to cut software turn-around time by 3 weeks (C++, Python, MATLAB)

## LEWG and Embedded Systems Working Group, C++ Standards Committee`

### Volunteer, October 2020 – Present

- Reviewed and provided feedback and insights about the design and challenges encountered with the implementation of [`std::expected`](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0323r5.html) as experienced with the users of my STX library’s equivalent [`stx::Result`](https://basit.pro/STX/structstx_1_1Result.html)

## Goldman Sachs – London, United Kingdom

### Software Engineer (Payments Engineering), June 2022 – June 2023

- Migrated the payment metadata handling system to a decoupled REST-based micro-service, improving query times by 43% (Java, SQL, JDBC)

## Microsoft – Lagos, Nigeria

### Software Engineering Intern (Mixed Reality), October 2020 – August 2021

- Architected and implemented Microsoft Mesh's (Mixed Reality) Unreal Engine plugin for Hololens and Windows Desktop, Enabling Software Developers and Microsoft's Partners to easily onboard and integrate the Mesh platform into their products (Unreal Engine, C++, C#, MRTK - Mixed Reality Toolkit, Microsoft UWP)
- Onboarded and introduced Software Engineers, stakeholders, and Microsoft's Partners to the Unreal Engine Plugin
- Mentored 15 Students from the Microsoft Learn Cohort, Introducing them to the technology stack and design space of developing mixed-reality software, and provided feedback and guidance for their projects

## Goldman Sachs – London, United Kingdom

### Software Engineering Intern (Data Pipeline Engineering), July 2020 – August 2020

- Re-architectured and re-designed a process health and metrics collection and monitoring system, improving runtime from 45 minutes to 2 seconds (SLANG Language, SecDB Database System)

## Data Science Nigeria – Lagos, Nigeria

### Deep Learning and Robotics Intern, July 2019 – October 2019

- Implemented an [MPU6050 IMU driver firmware](https://github.com/lamarrr/MPU60X0) used in a robotic system for estimating position and orientation running on an STM32 ARM MCU (C++, C)
- Implemented tests for Memory Allocation and Interpreter Invocations in Tensorflow Lite's C++ API examples [PR # 32917](https://github.com/tensorflow/tensorflow/pull/32917/files)
- Built a mobile app and trained a low-latency quantized deep learning model for recognizing Nigerian hand gestures from manually curated and locally-sourced data sets (Dart & Python)
- Tutored 14 University of Abomey-Calavi Ph.D. students, introducing them to Machine Learning and Computer Vision

## Google Developer Students’ Club, University of Ilorin – Kwara, Nigeria

### Mentor, December 2018 – August 2020

- Mentored 89 students on Machine Learning with Python and Tensorflow in our workshops
- Fixed a tick-rate dependency bug in Google-Nest’s OpenThread firmware [PR #18](https://github.com/openthread/ot-rtos/pull/19) (C++, FreeRTOS)

## OOTW Inc. – London, United Kingdom

### Contract Software Developer, September 2017 – May 2019

- Implemented and deployed customer-service chatbots using Dialogflow’s text-to-intent engine (Python)
- Researched and Implemented machine learning models for detecting anomalies in traffic and orders on our shopping service backend
- Built desktop GUI with to aid log investigation and diagnosis (Qt5, C++)

# PERSONAL PROJECTS

## Ashura (Vulkan, C++, CMake, GLSL), 2022 - Present

Ashura is a 2D and 3D Data-oriented GUI and Game Engine, Graphics Backend-agnostic. Also contains:

- 2D Canvas API for rendering 2D GUI Widgets
- Graphics API Render Hardware Interface (Vulkan-Backend) with Automatic Resource Synchronization
- Reimplementation of standard data structures better suited for real-time constrained software: `std::vector`, `std::vector<bool>`, `std::string_view`, `std::map`
- Pass-oriented Renderer and Render Graph
- Memory allocators: Arena memory allocator, Over-aligned heap allocator

## STX (C++, CMake), 2019 - Present

Implemented a monadic error & optional-value and utilities library to make C++ development easier on platforms/domains where exceptions and memory allocations are unacceptable. Also contains:

- Async Task Scheduling library backed by a Static Thread Pool
- Memory management library
- Multithreaded reference-counted objects, etc.
- **Presented at**: [CppCon](https://www.youtube.com/watch?v=MpWtS_I_pJI), [MeetingCpp](https://www.youtube.com/watch?v=8CZhJa8UJk0&t=2s), [CppCast](https://www.youtube.com/watch?app=desktop&v=Z3t0BW-PuG4)

## Valkyrie (C++, Skia, OpenGL), 2020 - 2022

The predecessor of Ashura, Implemented a 2D Graphics Engine with a Skia Graphics Backend

## Barracuda (C++, CUDA)

Implemented a C++ deep learning library for GPU-accelerated inference of Neural Networks on NVIDIA GPUs.

## A.U.T.K (Python)

Implemented an audio data spectral feature extraction and augmentation library for machine-learning-based automatic speech recognition apps.

# OPEN-SOURCE CONTRIBUTIONS

## Tensorflow-Lite Micro

- [Implemented Pooling kernel for Tensorflow Lite CMSIS-NN backend](https://github.com/tensorflow/tensorflow/pull/34145)
- [Added int8 Support and Unit Tests for the Negate Kernel]()
- Added reference kernel fallback for the [fully_connected](https://github.com/tensorflow/tensorflow/pull/34168), [depthwise](https://github.com/tensorflow/tensorflow/pull/34167), and [conv](https://github.com/tensorflow/tensorflow/pull/34164) kernels

## NVIDIA CuDF and Rapids Cuda Memory Manager

- [Removed c-style pointer casts and redundant `reinterpret_cast`s in cudf::io](https://github.com/rapidsai/cudf/pull/6386)
- [Refactor error.hpp out of detail](https://github.com/rapidsai/rmm/pull/1439)
