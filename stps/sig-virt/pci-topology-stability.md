# Openshift-virtualization-tests Test plan

## **PCI Topology Stability — Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** N/A — new functional tests for an existing platform guarantee, not a feature delivery
- **Feature Tracking:** N/A — not a feature delivery
- **Epic Tracking:** [CNV-81270](https://issues.redhat.com/browse/CNV-81270)
- **Feature Maturity:**
  - DP: N/A
  - TP: N/A
  - GA: N/A — not a new feature; tests cover an existing platform guarantee
- **QE Owner(s):** Samuel Alberstein (@SamAlber)
- **Owning SIG:** sig-virt
- **Participating SIGs:** sig-virt

**Document Conventions (if applicable):**
- PCI device lines: The list of PCI devices visible inside the guest, where each line pairs a device address with its description (e.g. `00:01.0 Ethernet controller: Red Hat, Inc. Virtio 1.0 network device`). Compared before and after lifecycle operations to detect topology changes.

### **Feature Overview**

When a virtual machine boots, every virtual device (disk, network interface, controller,
memory balloon) is assigned a PCI bus address. Guest operating systems rely on these addresses
being stable across reboots and migrations. If addresses shift, the guest may fail to recognize
disks, network interfaces, or other devices, leading to application failures or data
unavailability. This STP covers new functional tests that also serve as
downstream regression coverage to ensure PCI topology remains stable across VM lifecycle
operations (restart, live migration, snapshot/restore) and CNV upgrades.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - *List the key D/S requirements reviewed:* PCI device addresses must remain stable across VM restart, live migration, snapshot/restore, and CNV upgrade.

- [x] **Understand Value and Customer Use Cases**
  - *Describe the feature's value to customers:* Customers depend on stable device addresses so the guest continues to recognize disks, network interfaces, and other devices after lifecycle operations. Address shifts cause data unavailability and application failures.
  - *List the customer use cases identified:*
    1. As a VM administrator, I want my VM's device addresses to remain unchanged after a restart so that the guest OS continues to recognize all devices.
    2. As a VM administrator, I want my VM's device addresses to remain unchanged after live migration so that the guest OS continues to recognize all devices.
    3. As a VM administrator, I want my VM's device addresses to remain unchanged after restoring from a snapshot so that the guest OS continues to recognize all devices.
    4. As a cluster administrator, I want my VMs' device addresses to remain unchanged after a CNV upgrade so that the guest OS continues to recognize all devices.

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
    - Observability: No dedicated observability tooling. The platform tracks topology versioning internally, but this is not exposed in dashboards or metrics.
    - Documentation: No downstream customer-facing documentation exists — PCI address stability is an implicit platform guarantee rather than a documented feature.
    - Performance: No performance targets; address assignment is a one-time operation during VM startup with negligible overhead.
    - Security: No security implications.
    - Scalability: PCI topology assignment is per-VM. Tests verify single-VM address stability. Platform migration concurrency limits exist but are out of scope for this STP.
  - *Note any NFRs not covered and why:* UI/Usability: No user-facing interface — PCI topology is assigned automatically with no user configuration or interaction; no usability testing applies.

#### **2. Known Limitations**

- **PCI topology stability applies only to amd64 and arm64 architectures.** It is not supported on s390x and ppc64le, which use different bus topologies.
  - *Sign-off:* Michael Henriksen (@mhenriks)

- **PCI stability is not guaranteed for hotplugged disks across reboots.** A hotplugged disk may receive a different PCI address when the VM is rebooted. Confirmed by the feature developer.
  - *Sign-off:* Michael Henriksen (@mhenriks)

#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - *Key takeaways and concerns:*
    - No formal kickoff — these tests cover an existing platform guarantee rather than a new feature. Fix and upstream documentation reviewed independently with the feature developer.
    - Hotplugged disks may receive different PCI addresses after a reboot — tests must not include hotplug-then-reboot scenarios.
    - PCI topology stability applies only to amd64 and arm64 architectures; s390x and ppc64le use different bus topologies and are excluded from testing.
    - Verification must be done from inside the guest; host-side metadata exists but does not guarantee the guest sees the same layout.

- [x] **Technology Challenges**
  - *List identified challenges:* Guest must be reachable and have device-listing tools installed; images without these tools cannot be tested.
  - *Impact on testing approach:* Tests depend on guest connectivity and standard RHEL images that include the required tools.

- [x] **API Extensions**
  - *List new or modified APIs:* No new user-facing APIs. Topology versioning is managed internally by the platform.
  - *Testing impact:* Internal versioning is covered by upstream tests; downstream tests focus on the user-visible outcome (stable addresses).

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
and CNV upgrades. Verification is done by capturing PCI device lines
from the guest before and after operations and comparing them.

**Test VM Configuration:** A standard RHEL VM already includes a broad mix of PCI
device types (e.g., virtio network and block devices, chipset controllers, PCIe
root ports, memory balloon) without requiring a custom specification. No special
VM configuration is needed — the default topology provides sufficient device
diversity to exercise PCI address stability.

**Testing Goals**

- **[P0]** On amd64 and arm64, boot a RHEL VM, capture PCI device lines from inside the guest, stop and start the VM, recapture, and confirm every address+description pair is unchanged.
- **[P0]** On a multi-worker amd64 or arm64 cluster, boot a RHEL VM, capture PCI device lines from inside the guest, live-migrate the VM to another worker, recapture, and confirm every address+description pair is unchanged.
- **[P0]** On snapshot-capable storage, boot a RHEL VM, capture PCI device lines from inside the guest, take a snapshot and restore, recapture, and confirm every address+description pair is unchanged.
- **[P0]** Capture PCI device lines from inside each RHEL upgrade-lane VM before a CNV upgrade, complete the upgrade, recapture, and confirm every address+description pair is unchanged.

**Out of Scope (Testing Scope Exclusions)**

- **Internal topology versioning**
  - *Rationale:* The platform tracks topology versions internally. Versioning correctness (which version is assigned, how versions behave differently) is fully covered by upstream functional tests.
  - *PM/Lead Agreement:* Michael Henriksen (@mhenriks)

- **Backward compatibility between topology versions**
  - *Rationale:* Fully covered by upstream functional tests — existing VMs keep their assigned topology version and address layout.
  - *PM/Lead Agreement:* Michael Henriksen (@mhenriks)

- **PCI stability after hotplug and reboot**
  - *Rationale:* PCI stability is not guaranteed for hotplugged disks across reboots. Confirmed by feature developer.
  - *PM/Lead Agreement:* Michael Henriksen (@mhenriks)

- **Windows-specific PCI verification**
  - *Rationale:* Validation uses RHEL guests only. Windows PCI verification is excluded here as a QE scope decision.
  - *PM/Lead Agreement:* Denys Shchedrivyi (@dshchedr)

- **s390x and ppc64le architectures**
  - *Rationale:* These architectures use different bus topologies. PCI topology stability does not apply to them.
  - *PM/Lead Agreement:* Michael Henriksen (@mhenriks)

- **Negative/failure-path scenarios (e.g., behavior when PCI addresses shift)**
  - *Rationale:* This STP verifies addresses remain stable. There is no user-facing error handling or recovery mechanism to test; if addresses shift, it is a platform bug surfaced by these tests. No degraded-mode or fallback behavior exists for the user.
  - *PM/Lead Agreement:* Michael Henriksen (@mhenriks)

**Test Limitations**

- **PCI device enumeration depends on device-listing tools being available in the guest image.** The standard RHEL images include these tools.
  - *Sign-off:* Denys Shchedrivyi (@dshchedr)

- **Snapshot/restore tests require a storage class that supports volume snapshots.**
  - *Sign-off:* Denys Shchedrivyi (@dshchedr)

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** — Validates that the feature works according to specified requirements and user stories
  - *Details:* Validates that PCI device addresses remain stable across VM lifecycle operations (restart, migration, snapshot/restore) and CNV upgrades. Each test captures PCI device lines from the guest before and after the operation and asserts they match.

- [x] **Automation Testing** — Confirms test automation plan is in place for CI and regression coverage (all tests are expected to be automated)
  - *Details:* Lifecycle tests in `tests/virt/node/general/`, upgrade test in `tests/virt/upgrade/`.

- [x] **Regression Testing** — Verifies that new changes do not break existing functionality
  - *Details:* After merge, these tests provide regression coverage in standard Tier 2 CI. Upgrade test runs in the upgrade CI lane.

- [ ] **Self-Validation Testing** — Should any of the new tests be included in the self-validation test package?
  - *Details:* N/A — PCI topology stability is not a core operational scenario for the self-validation package.

**Non-Functional**

- [ ] **Performance Testing** — Validates feature performance meets requirements (latency, throughput, resource usage)
  - *Details:* N/A — No performance impact; topology assignment is a one-time operation during VM startup.

- [ ] **Scale Testing** — Validates feature behavior under increased load and at production-like scale
  - *Details:* N/A — PCI topology assignment is per-VM with no additional scale constraints. Platform migration concurrency limits are out of scope for this STP.

- [ ] **Security Testing** — Verifies security requirements, RBAC, authentication, authorization, and vulnerability scanning
  - *Details:* N/A — No RBAC surface or security implications.

- [ ] **Usability Testing** — Validates user experience and accessibility requirements
  - *Details:* N/A — No user-facing interface — PCI topology is assigned automatically with no user configuration or interaction.

- [ ] **Monitoring** — Does the feature require metrics and/or alerts?
  - *Details:* N/A — No new metrics or alerts required.

**Integration & Compatibility**

- [x] **Compatibility Testing** — Ensures feature works across supported platforms, versions, and configurations
  - *Details:* Tests run on both amd64 and arm64 clusters as part of sig-virt's standard CI. No dedicated multiarch (cross-architecture) tests or scheduled lanes — architecture coverage comes from existing CI topology.

- [x] **Upgrade Testing** — Validates upgrade paths from previous versions, data migration, and configuration preservation
  - *Details:* Dedicated upgrade test captures PCI device lines before and verifies they are unchanged after upgrade.

- [x] **Dependencies** — Blocked by deliverables from other components/products
  - *Details:* PCI topology stability depends on upstream platform logic; snapshot tests depend on the storage operator.

- [x] **Cross Integrations** — Does the feature affect other features or require testing by other teams?
  - *Details:* Snapshot/restore scenario depends on the storage operator for volume snapshot support. Storage operator availability is an environment prerequisite, not a cross-SIG test responsibility.

**Infrastructure**

- [ ] **Cloud Testing** — Does the feature require multi-cloud platform testing?
  - *Details:* N/A — Bare metal with RWX storage is the standard test environment.

#### **3. Test Environment**

- **Cluster Topology:** 3-master/3-worker bare-metal (2 workers minimum for migration tests)
- **OCP & OpenShift Virtualization Version(s):** OCP 4.22 and later with OpenShift Virtualization 4.22 and later (topology stability fix is available in 4.22; lifecycle tests validate address stability on any version, and the upgrade test validates that addresses are preserved across the upgrade)
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
- [x] PCI topology stability fix is available and functional in the target OpenShift Virtualization version
- [x] Snapshot support is available (for snapshot/restore scenario)

#### **5. Risks**

**Timeline/Schedule**

- **Mitigation:** Tests depend on the PCI topology stability fix (already merged) and ODF for snapshot storage (standard infrastructure, already available in CI). No deliverables are blocking test development or execution.

**Test Coverage**

- **Mitigation:** Each captured line pairs a device address with its device description. A swap between devices with distinct descriptions is detected. A swap between two devices with the same description (for example two identical NICs) would not change the captured set and would not be detected. Test VMs use one device of each type, so that case does not arise in these scenarios.

**Test Environment**

- **Mitigation:** Standard bare-metal cluster with RWX storage is sufficient.

**Untestable Aspects**

- **Mitigation:** All scenarios can be reproduced in a standard test environment.

**Resource Constraints**

- **Mitigation:** Tests use standard infrastructure and require no special hardware.

**Dependencies**

- **Risk:** PCI topology behavior depends on upstream address-assignment logic. Upstream changes could reintroduce address shifts.
  - **Mitigation:** Monitor upstream changes to address assignment. These tests are the downstream check that address shifts have not been reintroduced.
  - *Dependent teams or components:* Upstream platform compute stack.
  - *Sign-off:* Denys Shchedrivyi (@dshchedr)

---

### **III. Test Scenarios & Traceability**

- **[CNV-16326]** — As a VM administrator, I want my VM's device addresses to remain stable after restart.
  - *Test Scenario:* [Tier 2] Boot a VM, capture PCI device lines, stop and start the VM, wait until the VM is Running and the guest is reachable, capture PCI device lines again, verify they match.
  - *Priority:* P0

- **[CNV-16327]** — As a VM administrator, I want my VM's device addresses to remain stable after live migration.
  - *Test Scenario:* [Tier 2] Boot a VM, capture PCI device lines, live-migrate the VM to another node, wait until migration completes and the VM is Running on the target node, capture PCI device lines again, verify they match.
  - *Priority:* P0

- **[CNV-16328]** — As a VM administrator, I want my VM's device addresses to remain stable after snapshot restore.
  - *Test Scenario:* [Tier 2] Boot a VM, capture PCI device lines, take a snapshot, restore the VM from the snapshot, wait until the restored VM is Running and the guest is reachable, capture PCI device lines again, verify they match.
  - *Priority:* P0

- **[CNV-16329]** — As a cluster administrator, I want my VMs' device addresses to remain stable after a CNV upgrade.
  - *Test Scenario:* [Tier 2] Capture PCI device lines for all upgrade VMs before CNV upgrade, perform the upgrade, wait until the upgrade completes and cluster health checks pass, capture PCI device lines again, verify they match.
  - *Priority:* P0

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

* **Reviewers:**
  - QE Architect (OCP-V): [Ruth Netser](@rnetser)
  - QE Members (OCP-V): [Akriti Gupta](@akri3i), [Samuel Alberstein](@SamAlber)
  - Principal QE (OCP-V): [Denys Shchedrivyi](@dshchedr), [Vasiliy Sibirskiy](@vsibirsk)
  - Principal Developer (OCP-V): [Jed Lejosne](@jean-edouard), [Michael Henriksen](@mhenriks)
  - Product Manager/Owner: [Martin Tessun](@mtessun)

* **Approvers:**
  - QE Architect (OCP-V): [Ruth Netser](@rnetser)
  - Principal QE (OCP-V): [Denys Shchedrivyi](@dshchedr), [Vasiliy Sibirskiy](@vsibirsk)
  - Principal Developer (OCP-V): [Michael Henriksen](@mhenriks)
  - Product Manager/Owner: [Martin Tessun](@mtessun)
