# Goal
  + Allow topology to be specified for both pre-provisioned and dynamic provisioned PV so scheduler
    can correctly place a Pod using a volume to an appropriate node
  + Support aribitrary PV topology domain without modify in-tree code
  + Allow cooperation between scheduler and external-provisioner
  + Allow administrator to restrict allowed topologies per StorageClass

# Resource update
  + add NodeAffinity to PV 
  + add AllowedTopology to storage class 

# CSI 
  * IdentityServer: 
    GetPluginCapabilities: csi.PluginCapability_Service_VOLUME_ACCESSIBILITY_CONSTRAINTS
  * NodeServer:
    NodeGetInfo: NodeId and AccessibleTopology
    kubelet will create CSINodeInfo, and add labels to node object using topology
    "csi.kubernetes.io/topology.example.com_rack": "rack1"
  * ControllerServer:
    if bindmode of storageclass is WaitForFirstConsume, scheduler will pick one node
    so the selectedNode and allowedTopologies in storageclass will be passed to controller server.
    CreateVolume: return AccessibleTopology in response, which will be translated to NodeAffinity
    of the PV created by external-provisioner
  * Scheduler:
    During predicate phase, if a Pod reference a PVC that is bound to PV
    with NodeAffinity, NodeSelector will be evaluated against the node's label to
    filter the nodes that the Podd can be scheduled to.
  * kubelet:
    when mount a PV, NodeAffinity is verified against the Node when mouting the PVs.
