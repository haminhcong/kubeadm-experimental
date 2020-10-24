# K8S Kubeadm Experimental Playbook

```bash
ansible-playbook -i inventory.ini site.yml --tags "docker" -e ansible_ssh_user=centos --key-file "/PATH_TO_GOOGLE_CLOUD_VM_KEY" --tags "install_docker"
```
