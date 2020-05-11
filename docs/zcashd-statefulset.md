# Zcashd statefuleset

The statefulset will be deployed with Tekton Triggers.

The zcashd instances will be of 2 types.
- miners
- wallets

A statefulset will utilize 4 confimaps.
- Base for shared zcashd settings between all instances of zcashd in the box
- Statefulset specific configmap which will indicate which binary pods should use. This is dynamic and no pre existing template will exist.
- Miner configmap for miner specific settings
- Wallet configmap for wallet specific settings

Both use the same base configmap, as well as a secondary configmap to configure the role differences.

To start a stateful set the following parameters are required:
- binary name

The following are options
- number of miners
- number of wallets
- blockchain snapshot

Options are passed to the stateful set.
Binary name is passed to the pod as a parameter.