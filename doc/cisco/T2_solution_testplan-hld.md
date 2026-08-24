# T2 RDMA Solution Test Plan Scoping 

**Document Version:** 1.0  
**Date:** December 17, 2025  
**Author:** [Raghavendran Ramanathan/RDMA Team]  
**Project Name:** [SONiC T2 RDMA Solution Test Plan]

---

## 1. Introduction

### 1.1 Purpose
This document aims to define the scope, objectives, and approach for the testing activities related to the SONiC T2 RDMA Solution Testing, as listed in the [Solutions Testplan| https://cisco.sharepoint.com/:x:/r/sites/SONIC_on_SF/Shared%20Documents/SONiC-Test/T2%20CHO%20Results/SONiC_Solution_TestPlan.xlsx?d=we619b71ac4bd497891a3a52ad9c6f6fe&csf=1&web=1&e=GNOeME], hereafter referred to as the TestPlan in this document.

### 1.2 Scope of Document
This document covers:
*  the scope of testing and automation,
*  efforts needed and entry/exit criteria.

This documetn doesn't cover:
*  detailed test cases,
*  specific test data,
*  efforts needed for non RDMA cases,
*  details of SONiC in general.

### 1.3 References

*   `SONiC_Solution_TestPlan.xlsx` - [https://cisco.sharepoint.com/:x:/r/sites/SONIC_on_SF/Shared%20Documents/SONiC-Test/T2%20CHO%20Results/SONiC_Solution_TestPlan.xlsx?d=we619b71ac4bd497891a3a52ad9c6f6fe&csf=1&web=1&e=GNOeME]

## 2. Project Overview

### 2.1 Project Description
T2 RDMA Solution testing covers the RDMA related feauters - ECN, PFC and PFCWD - and their testcases under the purview of the SONiC solution. The testcases have been detailed in the TestPlan itself. Here we discuss the scope of execution and automation.

### 2.2 Key Stakeholders
Identify key individuals or groups involved in the project and testing (e.g., Product Management, Development, QA, Operations).
Product Management | Krithika Srinivas
Executive Sponsor | Mani Veerachamy
Test Manager | Sonum Mathur
Dev Lead | Alpesh Patel
Others | Zhixin Zhu, Dan Caugherty


## 3. Test Scope

### 3.1 In-Scope Items
List of the features, functionalities, modules, or areas of the SONiC solution that WILL be tested.
    *   ECN(Explicit Congestion Notification
    *   PFC|Priority Flow Control
    *   PFCWD|PFC watchdog feature
    *   Serviceability CLIs
    *   Routing protocols and traffic over PFC enabled ports
We will work with the following platforms:
    *   Vanguard(88-LC0-36FH-M) - 36 port 400G LC with Macsec and HBM.
    *   Lancer(88-LC0-36FH) - 36 port 400G LC

### 3.2 Out-of-Scope Items
Clearly list the features, functionalities, modules, or areas that WILL NOT be tested in this phase, along with a brief justification.

    *   Performance testing under extreme load (to be covered in a separate phase)
    *   Security vulnerability scanning (handled by a dedicated security team)
    *   Integration with third-party tools not explicitly mentioned in requirements.
    *   Any feature not listed in the In-Scope Items.

## 4. Test Objectives
What are the main goals of this testing effort?

    *   To verify that the T2 RDMA SONiC product meets all customer requirements.
    *   To ensure the stability and reliability of the solution on target hardware listed above.
    *   To identify and report defects prior to release.

## 5. Test Strategy and Approach

### 5.1 Test Types
We will be focused on solution level testing, which includes:
  * end-to-end protocol and traffic with or without congestion in the testbed.
  * Stability and recovery of traffic and routing under transient conditions like chassis reboot, interface-flap, LC reboot etc.

### 5.2 Test Environment
The tests will be conducted and automated using 
 * the either the solutions testbed as listed in the TestPlan, or
 * the Ixia-T2 testbeds, as listed in https://ciscoteams.atlassian.net/wiki/spaces/WHITEBOX/pages/730696328/T2-Ixia

### 5.3 Test Data Management
All test results will be uploaded to a xls sheet that is present in the TestPlan in a seperate column in the RDMA sheet.

### 5.4 Automation Strategy (if applicable)
We will automate tests in spytest using the framework provided in this PR: https://wwwin-github.cisco.com/whitebox/sonic-test/pull/1411
All efforts will be made to avoid duplicating a test in snappi or spytest.

## 6. Roles and Responsibilities
Define who is responsible for what aspects of the testing process.
*   **Test Lead and Engineer:** [Raghavendran Ramanathan/RDMA] - Overall test planning, strategy, execution, automation and reporting.
*   **Test Engineer:** [To Be Hired] - Exceution, automation and reporting.
*   **Development Team:** [RDMA Team] - Defect fixing, support.
*   **Product Management:** [Krithika Srinivas] - Requirements clarification, acceptance.

## 7. Entry and Exit Criteria

### 7.1 Test Entry Criteria
Conditions that must be met before formal testing can begin.
    *   All in-scope features are code complete and deployed to the test environment.
    *   Critical defects from previous cycles are resolved.
    *   Test environment is stable and ready.
    *   All tests that are already automated have been identified, and marked accordingly in the TestPlan.
    *   Test plan and test cases are reviewed and approved.

### 7.2 Test Exit Criteria
Conditions that must be met for testing to be considered complete.
    *   All high-priority test cases executed with a pass rate of 90%.
    *   All critical and major defects resolved or accepted with a workaround.
    *   Test coverage achieved (e.g., 90% of requirements).
    *   Final test report submitted and approved.

## 8. Deliverables
List the documents and artifacts that will be produced during the testing effort.

    *   Test Plan Scoping Document (this document)
    *   Detailed Test Cases in the TestPlan
    *   Defect Reports in JIRA. Will be labelled as T2-Prod.
    *   Test Execution Reports - will be initially marked in the TestPlan, and later a seperate xls in sharepoint.
    *   Test Summary Report

## 9. Tools and Infrastructure
List the tools that will be used for testing.

    *   Test Management Tool: Jira
    *   Defect Tracking Tool: Jira
    *   Automation Frameworks: [Pytest, Spytest, Snappi]
    *   Version Control: Git
    *   Traffic Genearators: Ixia, Spirent

## 10. Risks and Mitigation
Identify potential risks to the testing effort and plans to mitigate them.

    *   **Risk:** Unstable test environment.  
           **Mitigation:** Dedicated environment team, regular health checks.
    *   **Risk:** Insufficient resources/time. 
           **Mitigation:** Prioritize test cases.
    *   **Risk:** Scope creep.  
           **Mitigation:** Strict change control process, regular scope reviews.
