# Swimcheck
- This is a basic K8s deployment running a bash script in an alpine container to hit an api to check for swim lesson availability, and trigger a push notification if spaces come up.
> The Saturday class is really hard to get into...
- TODO:
  - Convert the deployment into a cronjob object instead and do away with the infinite while loop.