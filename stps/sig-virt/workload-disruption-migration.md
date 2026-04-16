# Openshift-virtualization-tests Test plan

## **Allow Workload Disruption Migration - Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** [kubevirt/kubevirt#11833](https://github.com/kubevirt/kubevirt/pull/11833)
- **Feature Tracking:** https://redhat.atlassian.net/browse/VIRTSTRAT-244
- **Epic Tracking:** https://redhat.atlassian.net/browse/CNV-54933
- **Feature Maturity:**
  - DP: N/A
  - TP: 4.19
  - GA: TBD
- **QE Owner(s):** Samuel Alberstein (@SamAlber)
- **Owning SIG:** sig-virt
- **Participating SIGs:** sig-virt

**Document Conventions:**

- AWD: Allow Workload Disruption — a migration policy option that permits the platform to pause a VM or switch to PostCopy to complete migration when standard live migration alone cannot converge.
- PostCopy: Migration mode where, after pre-copy fails to converge, the VM switches to the target node and remaining memory is fetched on-demand from the source.
- Paused: Migration mode where the VM is briefly paused to allow the final memory transfer, then resumed on the target node.

### **Feature Overview**

When a VM is live-migrated, the migration may fail to complete if the guest is writing to memory faster than
the platform can transfer it (high dirty rate). With the Allow Workload Disruption (AWD) migration policy, the platform is
permitted to use potentially disruptive migration modes (PostCopy or Paused) to help the migration converge
when pre-copy cannot. This is critical for operations that require migration — such as node drain
and CPU/memory hotplug — where a non-converging migration would block the operation entirely.
This allows maintenance operations to complete and in-guest processes to be preserved after migration.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

This section documents the mandatory QE review process. The goal is to understand the feature's value,
technology, and testability before formal test planning.

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - *List the key D/S requirements reviewed:* AWD migration policy enables potentially disruptive migration modes when standard live migration cannot converge within the configured completion timeout. When PostCopy is allowed, the VM starts on the target while remaining memory is fetched on-demand. Otherwise, the VM is briefly paused to allow the final memory transfer, then resumed on the target node.

- [x] **Understand Value and Customer Use Cases**
  - *Describe the feature's value to customers:* Customers need migration to converge for maintenance operations (node drain, hotplug) even at the cost of brief disruption, rather than having migrations fail and block the operation entirely.
  - *List the customer use cases identified:*
    1. As an admin, I want to migrate a VM with AWD policy so that migration completes in PostCopy or Paused mode when pre-copy cannot converge, without terminating running guest workloads.
    2. As an admin, I want to drain a node for maintenance so that VMs with AWD policy migrate and resume without terminating running guest workloads.
    3. As an admin, I want to hotplug CPU/memory to a running VM so that AWD migration completes and the guest reflects the new resources without terminating running guest workloads.

- [x] **Testability**
  - *Note any requirements that are unclear or untestable:* Testable by configuring an AWD migration policy with a tight completion timeout and capped bandwidth, then verifying the migration mode after migration.

- [x] **Acceptance Criteria**
  - *List the acceptance criteria:*
    1. When pre-copy migration does not converge, migration completes successfully in the expected mode (PostCopy or Paused) under AWD policy, regardless of migration trigger or guest operating system.
    2. After migration completes, workloads running in the guest before migration remain active without requiring guest reboot or application restart.

- [x] **Non-Functional Requirements (NFRs)**
  - *List applicable NFRs and their targets:* Monitoring: No new metrics or alerts required. Observability: No new observability requirements; migration mode is visible via existing status fields. UI: No UI component. Documentation: Covered by upstream and product documentation. Performance: No performance targets defined. Security: RBAC covered by core KubeVirt tests. Scalability: No scalability concerns; feature is per-VM.

#### **2. Known Limitations**

The limitations are documented to ensure alignment between development, QA, and product teams.
The following are confirmed product constraints accepted before testing begins.

- **s390x does not support memory hotplug, so memory hotplug scenarios are not applicable on this architecture.**
  - *Sign-off:* Martin Tessun / 2026-06-30

- **PostCopy mode is silently suppressed for VMs with VFIO devices (host devices, GPUs, SR-IOV interfaces); AWD falls back to Paused mode instead.**
  - *Sign-off:* Jed Lejosne / 2026-07-06

#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - *Key takeaways and concerns:* Feature reviewed through KubeVirt enhancement and design discussions.

- [x] **Technology Challenges**
  - *List identified challenges:* Triggering Paused mode reliably requires tuning bandwidth, completion timeout, and guest memory load to ensure standard (pre-copy) live migration does not converge before the timeout.

- [x] **API Extensions**
  - *List new or modified APIs:* New migration policy fields to control disruptive migration behavior and a status field to report the migration mode used.

- [x] **Test Environment Needs**
  - *See environment requirements in Section II.3 and testing tools in Section II.3.1*

- [x] **Topology Considerations**
  - *Describe topology requirements:* Requires at least 2 worker nodes for migration. Windows VMs require dedicated high-resource nodes.

### **II. Software Test Plan (STP)**

This STP serves as the **overall roadmap for testing**, detailing the scope, approach, resources, and schedule.

#### **1. Scope of Testing**

Tests validate that allow-workload-disruption (AWD) migration completes in the expected mode (PostCopy or Paused) across different migration triggers (explicit migration, CPU/memory hotplug) and guest operating systems (RHEL, Windows). Node drain is tested with RHEL guests only (see Out of Scope). Guest process preservation — verified by confirming a background process started before migration remains running after migration without restart — is checked after each migration.

**Testing Goals**

- **[P0]** AWD Migration Mode: Verify AWD migration falls back to PostCopy and Paused modes when pre-copy cannot converge, with process preservation.
- **[P0]** AWD Node Drain: Verify node drain triggers AWD migration in the expected mode with process preservation.
- **[P1]** AWD CPU Hotplug: Verify CPU hotplug triggers AWD migration and guest reports new CPU count with process preservation.
- **[P1]** AWD Memory Hotplug: Verify memory hotplug triggers AWD migration and guest reports new memory amount with process preservation.

**Out of Scope (Testing Scope Exclusions)**

The following items are explicitly Out of Scope for this test cycle and represent intentional exclusions.
No verification activities will be performed for these items, and any related issues found will not be classified as defects for this release.

- **Windows node drain**
  - *Rationale:* Node drain is a cluster-level operation that evicts VMs regardless of guest OS; the AWD migration path does not differ between RHEL and Windows during drain.
  - *PM/Lead Agreement:* Martin Tessun / 2026-06-30

**Test Limitations**

- **Triggering Paused mode reliably requires tuning bandwidth, completion timeout, and guest memory load to prevent standard live migration from converging before the timeout.**
  - *Sign-off:* Samuel Alberstein / 2026-06-30

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** — Validates that the feature works according to specified requirements and user stories

- [x] **Automation Testing** — Confirms test automation plan is in place for CI and regression coverage (all tests are expected to be automated)

- [x] **Regression Testing** — Verifies that new changes do not break existing functionality
  - *Details:* Tests validate both migration mode and hotplug functionality

- [ ] **Self-Validation Testing** — Should any of the new tests be included in the self-validation test package?
  - *Details:* N/A

**Non-Functional**

- [ ] **Performance Testing** — Validates feature performance meets requirements (latency, throughput, resource usage)
  - *Details:* Not in scope for functional AWD validation

- [ ] **Scale Testing** — Validates feature behavior under increased load and at production-like scale (e.g., large number of VMs, nodes, or concurrent operations)
  - *Details:* Scale testing is deferred; AWD is validated functionally per-VM. No scale-specific concerns identified.

- [ ] **Security Testing** — Verifies security requirements, RBAC, authentication, authorization, and vulnerability scanning
  - *Details:* N/A — Migration policy RBAC is covered by core KubeVirt tests

- [ ] **Usability Testing** — Validates user experience and accessibility requirements
  - *Details:* N/A — No UI component; migration mode is reported via standard status fields

- [ ] **Monitoring** — Does the feature require metrics and/or alerts?
  - *Details:* N/A — No specific metrics required for AWD

**Integration & Compatibility**

- [x] **Compatibility Testing** — Ensures feature works across supported platforms, versions, and configurations
  - *Details:* Parametrized across RHEL and Windows guest OSes
  - *Backward compatibility:* No known API changes affecting backward compatibility

- [ ] **Upgrade Testing** — Validates upgrade paths from previous versions, data migration, and configuration preservation
  - *Details:* Upgrade path evaluated; no AWD-specific upgrade concerns identified. Not in scope for this cycle.

- [x] **Dependencies** — Blocked by deliverables from other components/products. Identify what we need from other teams before we can test.
  - *Details:* Core AWD functionality is implemented in KubeVirt

- [x] **Cross Integrations** — Does the feature affect other features or require testing by other teams? Identify the impact we cause.
  - *Details:* AWD interacts with hotplug and node drain (sig-virt); tests cover migration triggered by both

**Infrastructure**

- [ ] **Cloud Testing** — Does the feature require multi-cloud platform testing? Consider cloud-specific features.
  - *Details:* N/A — Bare metal with RWX storage is required

#### **3. Test Environment**

- **Cluster Topology:** Bare Metal — Multi-worker OCP cluster (minimum 2 workers for migration, additional for Windows special_infra)
- **OCP & OpenShift Virtualization Version(s):** OCP 4.18+ — Feature available from OCP-V 4.18
- **CPU Virtualization:** VT-x / AMD-V
- **Compute Resources:** Standard + high-resource nodes — Standard workers for RHEL; high-resource workers (special_infra) for Windows VMs
- **Special Hardware:** N/A
- **Storage:** RWX default storage class (e.g., ocs-storagecluster-ceph-rbd-virtualization)
- **Network:** OVN-Kubernetes
- **Required Operators:** OpenShift Virtualization
- **Platform:** Bare Metal
- **Special Configurations:** N/A

#### **3.1. Testing Tools & Frameworks**

- **Test Framework:** Standard
- **CI/CD:** N/A
- **Other Tools:** N/A

#### **4. Entry Criteria**

The following conditions must be met before testing can begin:

- [x] Requirements and design documents are **approved and merged**
- [x] Test environment can be **set up and configured** (see Section II.3 - Test Environment)
- [x] AWD migration policy fields are available and functional in the target OCP-V version

#### **5. Risks**

**Timeline/Schedule**

- **Risk:** N/A
  - **Mitigation:** N/A

**Test Coverage**

- **Risk:** Paused mode may not reliably trigger if standard live migration converges too quickly on fast storage
  - **Mitigation:** Added guest memory pressure and tuned completion timeout to ensure migration does not converge before the disruptive mode triggers

**Test Environment**

- **Risk:** Windows tests require special infrastructure and high-resource nodes which may not be available in all CI environments
  - **Mitigation:** Tests are marked for CI lane selection so they only run in environments with the required infrastructure

**Untestable Aspects**

- **Risk:** Exact timing of PostCopy/Paused mode transition depends on cluster load, storage speed, and network conditions
  - **Mitigation:** Tests verify the final migration mode rather than transition timing

**Resource Constraints**

- **Risk:** N/A
  - **Mitigation:** N/A

**Dependencies**

- **Risk:** AWD behavior depends on KubeVirt migration engine implementation; changes in convergence logic could affect mode selection
  - **Mitigation:** Monitor KubeVirt upstream changes to migration convergence behavior

**Other**

- **Risk:** N/A
  - **Mitigation:** No additional risks identified

---

### **III. Test Scenarios & Traceability**

- **[CNV-15225]** — As an admin, I want AWD migration to complete in PostCopy mode when pre-copy cannot converge.
  - *Test Scenario:* [Tier 2] Migrate VM with AWD policy (PostCopy allowed); verify migration completes in PostCopy mode and background process is preserved.
  - *Priority:* P0

- **[CNV-15246]** — As an admin, I want AWD migration to complete in Paused mode when pre-copy cannot converge.
  - *Test Scenario:* [Tier 2] Migrate VM with AWD policy (PostCopy not allowed); verify migration completes in Paused mode and background process is preserved.
  - *Priority:* P0

- **[CNV-15245]** — As an admin, I want node drain to trigger AWD migration in the expected mode.
  - *Test Scenario:* [Tier 2] Drain the node hosting a VM with AWD policy; verify migration mode and process preservation after drain completes.
  - *Priority:* P0

- **[CNV-15234, CNV-15247]** — As an admin, I want CPU hotplug to trigger AWD migration and reflect the new CPU count in the guest.
  - *Test Scenario:* [Tier 2] Hotplug CPU sockets on VM with AWD policy; verify migration mode, guest CPU count, and process preservation.
  - *Priority:* P1

- **[CNV-15235, CNV-16312]** — As an admin, I want memory hotplug to trigger AWD migration and reflect the new memory amount in the guest.
  - *Test Scenario:* [Tier 2] Hotplug memory on VM with AWD policy; verify migration mode, guest memory amount, and process preservation.
  - *Priority:* P1

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

* **Reviewers:**
  - QE Architect (OCP-V): [Ruth Netser](@rnetser)
  - QE Members (OCP-V): [Akriti Gupta](@akri3i), [Samuel Alberstein](@SamAlber)
  - Principal QE (OCP-V): [Den Shchedrivyi](@dshchedr), [Vasiliy Sibirskiy](@vsibirsk)
  - Principal Developer (OCP-V): [Jed Lejosne](@jean-edouard)
  - Product Manager/Owner: [Martin Tessun](@mtessun)

* **Approvers:**
  - QE Architect (OCP-V): [Ruth Netser](@rnetser)
  - Principal QE (OCP-V): [Den Shchedrivyi](@dshchedr), [Vasiliy Sibirskiy](@vsibirsk)
