## Instance Creation

The following is the restful command to create the instance
```bash
POST https://www.googleapis.com/compute/v1/projects/<PROJECT>/zones/us-central1-a/instances
{
  "kind": "compute#instance",
  "name": "cvat-development",
  "zone": "projects/<PROJECT>/zones/us-central1-a",
  "machineType": "projects/<PROJECT>/zones/us-central1-a/machineTypes/n1-standard-4",
  "displayDevice": {
    "enableDisplay": false
  },
  "metadata": {
    "kind": "compute#metadata",
    "items": []
  },
  "tags": {
    "items": [
      "http-server",
      "https-server"
    ]
  },
  "disks": [
    {
      "kind": "compute#attachedDisk",
      "type": "PERSISTENT",
      "boot": true,
      "mode": "READ_WRITE",
      "autoDelete": true,
      "deviceName": "cvat-development",
      "initializeParams": {
        "sourceImage": "projects/ubuntu-os-cloud/global/images/ubuntu-1804-bionic-v20200923",
        "diskType": "projects/<PROJECT>/zones/us-central1-a/diskTypes/pd-standard",
        "diskSizeGb": "100"
      },
      "diskEncryptionKey": {}
    },
    {
      "kind": "compute#attachedDisk",
      "mode": "READ_WRITE",
      "autoDelete": false,
      "type": "PERSISTENT",
      "deviceName": "cvat-development-disk-1",
      "initializeParams": {
        "diskName": "cvat-development-disk-1",
        "diskType": "projects/<PROJECT>/zones/us-central1-a/diskTypes/pd-standard",
        "diskSizeGb": "1000",
        "description": "A 1 TB persistent disk for data storage.",
        "resourcePolicies": [
          "projects/<PROJECT>/regions/us-central1/resourcePolicies/cvat-daily-snapshot"
        ]
      }
    }
  ],
  "canIpForward": false,
  "networkInterfaces": [
    {
      "kind": "compute#networkInterface",
      "subnetwork": "projects/<PROJECT>/regions/us-central1/subnetworks/default",
      "accessConfigs": [
        {
          "kind": "compute#accessConfig",
          "name": "External NAT",
          "type": "ONE_TO_ONE_NAT",
          "natIP": "35.223.152.133",
          "networkTier": "PREMIUM"
        }
      ],
      "aliasIpRanges": []
    }
  ],
  "description": "",
  "labels": {},
  "scheduling": {
    "preemptible": false,
    "onHostMaintenance": "MIGRATE",
    "automaticRestart": true,
    "nodeAffinities": []
  },
  "deletionProtection": true,
  "reservationAffinity": {
    "consumeReservationType": "ANY_RESERVATION"
  },
  "serviceAccounts": [
    {
      "email": "199994669892-compute@developer.gserviceaccount.com",
      "scopes": [
        "https://www.googleapis.com/auth/cloud-platform"
      ]
    }
  ],
  "shieldedInstanceConfig": {
    "enableSecureBoot": false,
    "enableVtpm": true,
    "enableIntegrityMonitoring": true
  },
  "confidentialInstanceConfig": {
    "enableConfidentialCompute": false
  }
}
```

And the gcloud command line:

```bash
gcloud beta compute --project=<PROJECT> instances create cvat-development --zone=us-central1-a --machine-type=n1-standard-4 --subnet=default --address=35.223.152.133 --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=199994669892-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --tags=http-server,https-server --image=ubuntu-1804-bionic-v20200923 --image-project=ubuntu-os-cloud --boot-disk-size=100GB --boot-disk-type=pd-standard --boot-disk-device-name=cvat-development --create-disk="mode=rw,size=1000,type=projects/<PROJECT>/zones/us-central1-a/diskTypes/pd-standard,name=cvat-development-disk-1,description=A 1 TB persistent disk for data storage.,device-name=cvat-development-disk-1,disk-resource-policy=projects/<PROJECT>/regions/us-central1/resourcePolicies/cvat-daily-snapshot" --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud compute --project=<PROJECT> firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server

gcloud compute --project=<PROJECT> firewall-rules create default-allow-https --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:443 --source-ranges=0.0.0.0/0 --target-tags=https-server
```
