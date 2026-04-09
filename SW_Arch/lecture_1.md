### Definition of Software system architecture

```text
 The software architecture of a system is a high-level description of the system's structure, its different components,
 and how those components communicate with each other to fulfill the system's requirements and constraints
```

**Many different levels of abstraction**
- Classes/structs
- Modules/packages/libraries
- Services(process/group of processes)

**Software Development Cycle**
- `Design` <--- architecture phase
- `Implementation`
- `Testing`
- `Deployment`


### High level of Ambiguity(모호성)
- `System Design has high level of ambiguity`
- why?
  - The person providing the requirements is often not an engineer and may even be not very technical
  - Getting the requirements is part of the solution.


**Requirements Classification**
- Features of the System
  - Functional Requirements: `Describe the system behavior - what the system must do.`
- Quality Attributes (품질 속성: 신뢰성, 가용성, 확장성, 보안성, 성능 ...etc)
  - Non-Functional requirements
- System Constraints
  - Limitation and boundaries 

Those three also called `Architecture Drivers`

```text
# Example

When a rider logs into the service mobile app,                    # <--- INPUT 
the system must display a map with nearby within 5 miles radius   # <--- OUTPUT
```