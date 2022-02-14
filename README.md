# nomad-hcloud-autoscaler

Example configuration

`config.hcl`
```
plugin_dir = "/local/plugins/"

# https://www.nomadproject.io/tools/autoscaling/agent/http
http {
  bind_address = "0.0.0.0"
  bind_port    = {{ env "NOMAD_PORT_nomad_autoscaler" }}
}

# https://www.nomadproject.io/tools/autoscaling/agent/policy
policy {
  dir                         = "/local/policies"
  default_cooldown            = "5m"
  default_evaluation_interval = "10s"
}

# https://www.nomadproject.io/tools/autoscaling/agent/nomad
nomad {
  address     = "https://nomad.service.[[ server_dc ]].consul:4646"
  region      = "[[ server_region ]]"
  namespace   = "default"

  ca_cert     = "/secrets/nomad/ca.crt"
  client_cert = "/secrets/nomad/agent.crt"
  client_key  = "/secrets/nomad/agent.key"
  
  token       = "FILL_ME"
}

# https://www.nomadproject.io/tools/autoscaling/agent/telemetry
telemetry {
    prometheus_metrics = true
    disable_hostname   = true
}

# https://www.nomadproject.io/tools/autoscaling/agent/apm
apm "prometheus" {
  driver = "prometheus"
  config = {
    address = "http://{{ range service "prometheus" }}{{ .Address }}:{{ .Port }}{{ end }}"
  }
}

# https://www.nomadproject.io/tools/autoscaling/agent/strategy
strategy "target-value" {
  driver = "target-value"
}

# https://www.nomadproject.io/tools/autoscaling/agent/target
target "nomad-hcloud-autoscaler" {
  driver = "nomad-hcloud-autoscaler"
  config = {
    hcloud_token = "FILL_ME_IN"
  }
}
```

`policy`

```
# https://www.nomadproject.io/tools/autoscaling/policy
scaling "hcloud_cluster" {
  enabled = true
  min     = 1
  max     = 3

  policy {
    cooldown            = "1m"
    evaluation_interval = "10s"

    check "cpu_allocated_percentage" {
      source = "prometheus"
      query  = "nomad_client_allocated_cpu{node_class=\"nomad_clients\"} / (nomad_client_allocated_cpu{node_class=\"nomad_clients\"} + nomad_client_unallocated_cpu{node_class=\"nomad_clients\"}) * 100"

      # https://www.nomadproject.io/tools/autoscaling/plugins/strategy/target-value
      strategy "target-value" {
        target    = 70
        threshold = 0.01
      }
    }

    check "mem_allocated_percentage" {
      source = "prometheus"
      query  = "nomad_client_allocated_memory{node_class=\"nomad_clients\"} / (nomad_client_allocated_memory{node_class=\"nomad_clients\"} + nomad_client_unallocated_memory{node_class=\"nomad_clients\"}) * 100"

      strategy "target-value" {
        target    = 70
        threshold = 0.01
      }
    }

    target "nomad-hcloud-autoscaler" {
      dry-run       = false
      node_class    = "nomad_clients"
      datacenter    = "nbg1"

      hcloud_name_prefix = "nomad-client"
      hcloud_location    = "nbg1"
      hcloud_server_type = "cpx11"
      hcloud_image       = "snapshot_id"
      hcloud_user_data   = "not required"
      hcloud_ssh_keys    = "ssh key name or id"
      hcloud_networks    = "network_name"
      hcloud_labels      = "label1=label1,label2=label2"
      # hcloud_datacenter  = "not required. mutually exclusive with location"

      node_drain_deadline           = "15m"
      node_drain_ignore_system_jobs = true
      node_purge                    = true
      node_selector_strategy        = "least_busy"
    }
  }
}
```
