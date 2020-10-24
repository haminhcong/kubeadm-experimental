# K8S Kubeadm Experimental Playbook

```bash
ansible-playbook -i inventory.ini site.yml \
    --tags "docker" \
    -e ansible_ssh_user=${sshUserName} \
    --key-file "/path/to/google_cloud_key"
```
