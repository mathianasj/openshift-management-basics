# Install dev spaces

1. oc create -f 1__devspaces-operator.yaml
1. Wait for operator to become available
1. oc create -f 2__devspaces-che.yaml