# Openshift-virtualization-tests Test plan

## **PCI Topology Stability — Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** N/A — downstream regression tests for PCI topology stability
- **Feature Tracking:** N/A — regression tests, not a feature delivery
- **Epic Tracking:** [CNV-81270](https://issues.redhat.com/browse/CNV-81270)
- **Feature Maturity:**
  - DP: N/A
  - TP: N/A
  - GA: N/A — regression tests, not a new feature
- **QE Owner(s):** Samuel Alberstein (@SamAlber)
- **Owning SIG:** sig-virt
- **Participating SIGs:** sig-virt

**Document Conventions (if applicable):**
- PCI fingerprint: An md5 hash of the sorted PCI device addresses visible to the guest, used to detect topology changes across lifecycle operations.

### **Feature Overview**

When a virtual machine boots, every virtual device (disk, network interface, controller,
memory balloon) is assigned a PCI bus address. Guest operating systems rely on these addresses
being stable across reboots and migrations. If addresses shift, the guest may fail to recognize
disks, network interfaces, or other devices, leading to application failures or data
unavailability. This STP covers downstream regression tests
to ensure PCI topology remains stable across VM lifecycle operations (restart, live migration,
snapshot/restore) and CNV upgrades.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - *List the key D/S requirements reviewed:* PCI device addresses must remain stable across VM restart, live migration, snapshot/restore, and CNV upgrade.

- [x] **Understand Value and Customer Use Cases**
  - *Describe the feature's value to customers:* Customers depend on stable device addresses so the guest continues to recognize disks, network interfaces, and other devices after lifecycle operations. Address shifts cause data unavailability and application failures.
  - *List the customer use cases identified:*
    1. As a VM administrator, I want my VM's device addresses to remain unchanged after a restart so that the guest OS continues to recognize all devices.
    2. As a VM administrator, I want my VM's device addresses to remain unchanged after live migration so that applications continue to function.
    3. As a VM administrator, I want my VM's device addresses to remain unchanged after restoring from a snapshot so that applications resume with the same device layout.
    4. As a cluster administrator, I want my VMs' device addresses to remain unchanged after a CNV upgrade so that the upgrade does not disrupt running workloads.

- [x] **Testability**
  - *Note any requirements that are unclear or untestable:* None. All requirements are testable by capturing PCI device addresses from the guest before and after each operation.

- [x] **Acceptance Criteria**
  - *List the acceptance criteria:*
    - PCI device addresses visible to the guest are identical before and after a VM restart.
    - PCI device addresses visible to the guest are identical before and after a live migration.
    - PCI device addresses visible to the guest are identical before and after a snapshot restore.
    - PCI device addresses visible to the guest are identical before and after a CNV upgrade.
  - *Note any gaps or missing criteria:* None

- [x] **Non-Functional Requirements (NFRs)**
  - *List applicable NFRs and their targets:*
    - Monitoring: No new metrics or alerts required.
    - Observability: No dedicated observability tooling. Topology version can be inspected via a VM annotation, but this is not exposed in dashboards or metrics.
    - Documentation: Developer-facing design documentation exists upstream (`docs/pci-topology.md` in kubevirt/kubevirt). No downstream customer-facing documentation exists — this is an implicit guarantee rather than a documented feature.
    - Performance: No performance targets; topology assignment is a one-time operation during VM startup with negligible overhead.
    - Security: No security implications.
    - Scalability: Feature is per-VM; no scale concerns.
  - *Note any NFRs not covered and why:* UI/Usability: No user-facing interface — PCI topology is assigned automatically with no user configuration or interaction; no usability testing applies.

#### **2. Known Limitations**

- **PCI topology management only applies to machine types with PCIe bus hierarchies** (q35 on amd64, virt on arm64). It is skipped for s390x and ppc64le which use different bus topologies.
  - *Sign-off:* TBD

- **PCI stability is not guaranteed for hotplugged disks across reboots.** A disk added via hotplug (with persist option) may receive a different PCI address when the VM is rebooted. This is a known current limitation confirmed by a developer.
  - *Sign-off:* TBD

#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - *Key takeaways and concerns:* No formal kickoff — regression test story. Upstream fix and documentation reviewed independently. Test plan confirmed with developer; hotplug disk address instability across reboots confirmed as a known limitation.

- [x] **Technology Challenges**
  - *List identified challenges:* Verification requires running commands inside the guest OS to observe the assigned addresses — there is no host-side API that exposes the guest-visible topology.
  - *Impact on testing approach:* Tests must SSH into the guest and parse device enumeration output.

- [x] **API Extensions**
  - *List new or modified APIs:* No new user-facing APIs. Topology version is tracked internally by the platform.
  - *Testing impact:* Annotation presence and versioning are covered by upstream tests; downstream tests focus on the user-visible outcome (stable addresses).

- [x] **Test Environment Needs**
  - *See environment requirements in Section II.3 and testing tools in Section II.3.1*

- [x] **Topology Considerations**
  - *Describe topology requirements:* At least 2 worker nodes required for migration tests. Feature applies only to amd64 and arm64 architectures.
  - *Impact on test design:* Migration test requires multi-worker cluster. Restart and snapshot/restore tests work on any topology including SNO.

---

### **II. Software Test Plan (STP)**

This STP serves as the **overall roadmap for testing**, detailing the scope, approach, resources, and schedule.

#### **1. Scope of Testing**

Tests validate that PCI device addresses remain stable across VM lifecycle operations
and CNV upgrades. Verification is done by capturing a PCI fingerprint from the guest
before and after operations and comparing them.

**Testing Goals**

- **[P0]** As a VM administrator, I can restart my VM and all device addresses remain unchanged, so the guest OS continues to recognize all devices.
- **[P0]** As a VM administrator, I can live-migrate my VM to another node and all device addresses remain unchanged, so applications continue to function.
- **[P0]** As a VM administrator, I can restore my VM from a snapshot and all device addresses remain unchanged, so applications resume with the same device layout.
- **[P0]** As a cluster administrator, I can upgrade CNV and all VMs' device addresses remain unchanged, so the upgrade does not disrupt running workloads.

**Out of Scope (Testing Scope Exclusions)**

- **Topology version annotation presence**
  - *Rationale:* Fully covered by upstream functional tests (annotation set during VM creation and template processing).
  - *PM/Lead Agreement:* TBD

- **Backward compatibility (v2/v3)**
  - *Rationale:* Fully covered by upstream functional tests (v2 frozen slot count preserves addresses across restart, v2 produces different addresses than v3, annotation propagation during template processing).
  - *PM/Lead Agreement:* TBD

- **PCI stability after hotplug and reboot**
  - *Rationale:* PCI stability is not guaranteed for hotplugged disks across reboots. Confirmed by feature developer.
  - *PM/Lead Agreement:* TBD

- **Windows-specific PCI verification**
  - *Rationale:* PCI topology is assigned on the host side identically for all guest OSes. Linux verification is sufficient.
  - *PM/Lead Agreement:* TBD

- **s390x and ppc64le architectures**
  - *Rationale:* These architectures use different bus topologies. PCI topology management is explicitly skipped upstream.
  - *PM/Lead Agreement:* TBD

**Test Limitations**

- **PCI fingerprint verification depends on device enumeration tools being available in the guest image.** The standard RHEL and Fedora images include these tools.
  - *Sign-off:* TBD

- **Snapshot/restore tests require a storage class that supports volume snapshots.**
  - *Sign-off:* TBD

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** — Validates that the feature works according to specified requirements and user stories
  - *Details:* Validates that PCI device addresses remain stable across VM lifecycle operations (restart, migration, snapshot/restore) and CNV upgrades. Each test captures a PCI fingerprint before and after the operation and asserts they match.

- [x] **Automation Testing** — Confirms test automation plan is in place for CI and regression coverage (all tests are expected to be automated)
  - *Details:* Lifecycle tests in `tests/virt/node/general/`, upgrade test in `tests/virt/upgrade/`.

- [x] **Regression Testing** — Verifies that new changes do not break existing functionality
  - *Details:* These tests are themselves regression guards. Run as part of standard Tier 2 CI. Upgrade test runs in upgrade CI lane.

- [ ] **Self-Validation Testing** — Should any of the new tests be included in the self-validation test package?
  - *Details:* N/A — PCI topology stability is a regression guard, not a core operational scenario for self-validation.

**Non-Functional**

- [ ] **Performance Testing** — Validates feature performance meets requirements (latency, throughput, resource usage)
  - *Details:* N/A — No performance impact; topology assignment is a one-time operation during VM startup.

- [ ] **Scale Testing** — Validates feature behavior under increased load and at production-like scale
  - *Details:* N/A — Feature is per-VM; no scale concerns.

- [ ] **Security Testing** — Verifies security requirements, RBAC, authentication, authorization, and vulnerability scanning
  - *Details:* N/A — No RBAC surface or security implications.

- [ ] **Usability Testing** — Validates user experience and accessibility requirements
  - *Details:* N/A — No user-facing interface — PCI topology is assigned automatically with no user configuration or interaction.

- [ ] **Monitoring** — Does the feature require metrics and/or alerts?
  - *Details:* N/A — No new metrics or alerts required.

**Integration & Compatibility**

- [ ] **Compatibility Testing** — Ensures feature works across supported platforms, versions, and configurations
  - *Details:* N/A — Tests are architecture-agnostic and run on both amd64 and arm64 clusters as part of sig-virt's standard CI (`--cpu-arch=arm64`). No dedicated multiarch (cross-architecture) tests or scheduled lanes.

- [x] **Upgrade Testing** — Validates upgrade paths from previous versions, data migration, and configuration preservation
  - *Details:* Dedicated upgrade test captures PCI fingerprints before and verifies they are unchanged after upgrade.

- [x] **Dependencies** — Blocked by deliverables from other components/products
  - *Details:* Core PCI topology logic is implemented upstream; snapshot depends on the storage operator.

- [x] **Cross Integrations** — Does the feature affect other features or require testing by other teams?
  - *Details:* Snapshot/restore scenario depends on the storage operator for volume snapshot support. Storage operator availability is an environment prerequisite, not a cross-SIG test responsibility.

**Infrastructure**

- [ ] **Cloud Testing** — Does the feature require multi-cloud platform testing?
  - *Details:* N/A — Bare metal with RWX storage is the standard test environment.

#### **3. Test Environment**

- **Cluster Topology:** 3-master/3-worker bare-metal (2 workers minimum for migration tests)
- **OCP & OpenShift Virtualization Version(s):** OCP 4.22 and later with OpenShift Virtualization 4.22 and later (v3 topology fix lands in 4.22; lifecycle tests validate a contract expected on any version, but the upgrade test specifically targets the v2→v3 transition)
- **CPU Virtualization:** VT-x / AMD-V — required for VM execution
- **Compute Resources:** Standard — no special compute requirements
- **Special Hardware:** N/A
- **Storage:** RWX default storage class; snapshot-capable storage class for snapshot/restore tests (e.g., ocs-storagecluster-ceph-rbd-virtualization)
- **Network:** OVN-Kubernetes (IP stack is not relevant — PCI topology is independent of network protocol)
- **Required Operators:** OpenShift Virtualization; ODF (OpenShift Data Foundation) for snapshot-capable storage class — environment prerequisite, not managed by this test plan
- **Platform:** Bare metal
- **Special Configurations:** N/A

#### **3.1. Testing Tools & Frameworks**

- **Test Framework:** Standard
- **CI/CD:** Lifecycle tests (restart, migration, snapshot/restore) run in the sig-virt Tier 2 lane. Upgrade test runs in the sig-virt upgrade lane.
- **Other Tools:** N/A

#### **4. Entry Criteria**

The following conditions must be met before testing can begin:

- [x] Requirements and design documents are **approved and merged**
- [x] Test environment can be **set up and configured** (see Section II.3 - Test Environment)
- [x] PCI topology fix is available and functional in the target CNV version
- [x] Snapshot APIs are available (for snapshot/restore scenario)

#### **5. Risks**

**Timeline/Schedule**

- **Mitigation:** Tests depend on the upstream PCI topology fix (already merged) and ODF for snapshot storage (standard infrastructure, already available in CI). No deliverables are blocking test development or execution.

**Test Coverage**

- **Risk:** PCI fingerprint captures only device addresses, not which specific device is at which address. A bug that swaps two addresses would not be detected.
  - **Mitigation:** Sufficient for regression detection. Exact device-to-address mapping is covered by upstream unit tests.
  - *Areas with reduced coverage:* Device-to-address mapping (covered upstream).
  - *Sign-off:* TBD

**Test Environment**

- **Mitigation:** Standard bare-metal cluster with RWX storage is sufficient.

**Untestable Aspects**

- **Mitigation:** All scenarios can be reproduced in a standard test environment.

**Resource Constraints**

- **Mitigation:** Tests use standard infrastructure and require no special hardware.

**Dependencies**

- **Risk:** PCI topology behavior depends on upstream device-address allocation logic. Upstream changes could reintroduce address shifts.
  - **Mitigation:** Monitor upstream changes to device-address allocation and hotplug port handling. These tests serve as the downstream regression guard.
  - *Dependent teams or components:* Upstream KubeVirt compute stack.
  - *Sign-off:* TBD

---

### **III. Test Scenarios & Traceability**

- **[CNV-16326]** — As a VM administrator, I want my VM's device addresses to remain stable after restart.
  - *Test Scenario:* [Tier 2] Boot a VM, capture PCI fingerprint, stop and start the VM, capture fingerprint again, verify they match.
  - *Priority:* P0

- **[CNV-16327]** — As a VM administrator, I want my VM's device addresses to remain stable after live migration.
  - *Test Scenario:* [Tier 2] Boot a VM, capture PCI fingerprint, live-migrate the VM to another node, capture fingerprint again, verify they match.
  - *Priority:* P0

- **[CNV-16328]** — As a VM administrator, I want my VM's device addresses to remain stable after snapshot restore.
  - *Test Scenario:* [Tier 2] Boot a VM, capture PCI fingerprint, take a snapshot, restore the VM from the snapshot, capture fingerprint again, verify they match.
  - *Priority:* P0

- **[CNV-16329]** — As a cluster administrator, I want my VMs' device addresses to remain stable after a CNV upgrade.
  - *Test Scenario:* [Tier 2] Capture PCI fingerprints for all upgrade VMs before CNV upgrade, perform the upgrade, capture fingerprints again, verify they match.
  - *Priority:* P0

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

* **Reviewers:**
  - QE Architect (OCP-V): [Ruth Netser](@rnetser)
  - QE Members (OCP-V): [Akriti Gupta](@akri3i), [Samuel Alberstein](@SamAlber)
  - Principal QE (OCP-V): [Den Shchedrivyi](@dshchedr), [Vasiliy Sibirskiy](@vsibirsk)
  - Principal Developer (OCP-V): [Jean-Edouard Babin](@jean-edouard)
  - Product Manager/Owner: [Martin Tessun](@mtessun)

* **Approvers:**
  - QE Architect (OCP-V): [Ruth Netser](@rnetser)
  - Principal QE (OCP-V): [Den Shchedrivyi](@dshchedr), [Vasiliy Sibirskiy](@vsibirsk)

**Sign-off checklist:**

- [x] Tier 1 / Tier 2 tests defined and traceability matrix updated.
- [ ] **Automation merged** (mandatory for GA).
- [ ] Tests running in release checklist jobs.
- [ ] Documentation reviewed.
