# **Relatório Técnico de TI**
## **Automação de Instalação do Cluster OpenShift (OCP) em Ambiente Power via Terraform**

---
# **Capa**
**Projeto:** Implementação de Cluster OpenShift em PowerVM via Terraform  
**Ambiente:** IBM Power10 / PowerVC / RHEL / RHCOS  
**Documento Base:** Automação OCP Power – Final fileciteturn0file0  
**Data:** 2025  
**Responsável:** *(preencher)*

---
# **Índice**
1. [Introdução](#introdução)
2. [Pré-requisitos](#pré-requisitos)
3. [Criação da LPAR RHEL 8.8](#criação-da-lpar-rhel-88)
4. [Configuração do CoreOS (RHCOS)](#configuração-do-coreos-rhcos)
5. [Criação das Imagens no PowerVC](#criação-das-imagens-no-powervc)
6. [Compute Templates](#compute-templates)
7. [Configuração de Rede no PowerVC](#configuração-de-rede-no-powervc)
8. [Configuração e Execução do Script Terraform](#configuração-e-execução-do-script-terraform)
9. [Detalhes do PowerVC](#detalhes-do-powervc)
10. [Detalhes do Cluster OpenShift](#detalhes-do-cluster-openshift)
11. [Configuração de Storage](#configuração-de-storage)
12. [Considerações Finais](#considerações-finais)

---

# **1. Introdução**
Este documento apresenta o processo completo para criação de um cluster OpenShift Container Platform (OCP) em infraestrutura IBM Power, utilizando PowerVC como gerenciador de virtualização e Terraform como ferramenta de automação.

O procedimento inclui a preparação das imagens de RHEL e RHCOS, criação das LPARs, templates de compute, configuração de rede e execução do script oficial **ocp4-upi-powervm**.

---

# **2. Pré-requisitos**
- PowerVC configurado e com acesso ao ambiente Power10.
- HMC com permissão para criação e gerenciamento de LPARs.
- Imagens ISO:
  - RHEL 8.8
  - CoreOS (RHCOS)
- Terraform instalado (recomendado uso com VS Code).
- Repositório GitHub:
  https://github.com/ocp-power-automation/ocp4-upi-powervm

---

# **3. Criação da LPAR RHEL 8.8**
1. Baixar imagem ISO do RHEL 8.8.
2. Na HMC, criar LPAR com:
   - **4 vCPUs**
   - **16 GB RAM**
   - **100 GB disco**
3. Instalar RHEL 8.8 e registrar no RedHat.
4. Atualizar para RHEL 8.10:
   - `yum update`
   - `yum upgrade`
5. Baixar a imagem ISO do CoreOS dentro do RHEL.
6. Criar usuário **core** e gerar chaves SSH (`ssh-keygen`).
7. Copiar chaves para o local onde ficará o script Terraform (`ocp4-upi-powervm-main/data`).
8. Instalar cloud-init:
   ```bash
   dnf install cloud-init
   ```
9. Instalar RMC (DynamicRM):
   ```bash
   wget https://public.dhe.ibm.com/software/server/POWER/Linux/yum/download/ibm-power-repo-latest.noarch.rpm
   rpm -ivh --nodeps ibm-power-repo-latest.noarch.rpm
   cd /opt/ibm/lop
   ./configure
   yum search rm
   yum install DynamicRM
   ```

Após esses passos, importar a LPAR no PowerVC (deve aparecer em **VM List**).

---

# **4. Configuração do CoreOS (RHCOS)**
1. Em **Available Volumes**, anexar:
   - Disco boot da LPAR RHEL.
   - Disco adicional de **120 GB**.
2. Na VM RHEL, instalar:
   ```bash
   yum install wget qemu-img parted gzip
   ```
3. Garantir que a ISO do CoreOS esteja acessível.
4. Descompactar a imagem:
   ```bash
   gunzip rhcos-openstack.ppc64le.qcow2.gz
   ```
5. Converter QCOW2 para RAW:
   ```bash
   qemu-img convert -f qcow2 -O raw rhcos-openstack.ppc64le.qcow2 rhcos-latest.raw
   ```
6. Identificar disco de 120GB (`lsblk`).
7. Copiar imagem RAW para o disco:
   ```bash
   dd if=rhcos-latest.raw of=${empty_disk_device} bs=4M
   ```
8. Desanexar o volume.

---

# **5. Criação das Imagens no PowerVC**
## Imagem do RHEL (Bastion)
Criada a partir da VM importada.

## Imagem do CoreOS
1. PowerVC → Images → Create.
2. Nome: **cores-img**.
3. Hypervisor: **PowerVM**.
4. Sistema operacional: **COREOS**.
5. Endianness: **littleEndian**.
6. Selecionar volume usado para imagem RAW.
7. Definir **Boot = Yes**.

---

# **6. Compute Templates**
Configurar templates para cada tipo de nó:

| Tipo | vCPUs | RAM | Disco |
|------|-------|------|--------|
| Bootstrap | 2 | 16 GB | 120 GB |
| Master | 2 | 32 GB | 120 GB |
| Worker | 2 | 32 GB | 120 GB |
| Bastion | 2 | 16 GB | 200 GB |

Observação: sistemas Power utilizam **SMT=8**, portanto **2 vCPUs = 16 CPUs lógicas**.

---

# **7. Configuração de Rede no PowerVC**
- A rede utilizada deve ter nome igual ao configurado no Terraform.
- DHCP será controlado pelo **Bastion Host**.
- Necessário definir apenas o range de IP.
- Ajustar DHCP do laboratório para ignorar MAC addresses definidos.

---

# **8. Configuração e Execução do Script Terraform**
No VS Code:

1. Editar o arquivo **var.tfvars**.
2. Executar:
   ```bash
   terraform init
   terraform apply -var-file var.tfvars
   ```
3. O script consumirá:
   - Imagens criadas no PowerVC
   - Compute templates
   - Configurações de rede e storage

---

# **9. Detalhes do PowerVC**
```tf
auth_url  = "https://192.168.140.115:5000/v3/>"
user_name = "root"
password  = "fit123"
tenant_name = "ibm-default"
domain_name = "Default"
openstack_availability_zone = "Default Group:902821B_78B51B1"
network_name = "openshift-p10"
```

### IDs das imagens
- RHEL Bastion: **ba1f2164-5c90-4be2-a23e-c5ed83a67d51**
- RHCOS nós: **e0f7fe48-c2fe-4a58-904b-7c00b82aad43**

---

# **10. Detalhes do Cluster OpenShift**
```tf
bastion   = { instance_type = "ocp.bastion",   image_id = "ba1f2164-5c90-4be2-a23e-c5ed83a67d51", count = 1, fixed_ip_v4 = "192.168.140.150" }
bootstrap = { instance_type = "ocp.bootstrap", image_id = "e0f7fe48-c2fe-4a58-904b-7c00b82aad43", count = 0, fixed_ips = ["192.168.140.151"] }
master    = { instance_type = "ocp.master",    image_id = "e0f7fe48-c2fe-4a58-904b-7c00b82aad43", count = 3, fixed_ips = ["192.168.140.152", "192.168.140.153", "192.168.140.154"] }
worker    = { instance_type = "ocp.worker",    image_id = "e0f7fe48-c2fe-4a58-904b-7c00b82aad43", count = 2, fixed_ips = ["192.168.140.155", "192.168.140.156"] }
```

---

# **11. Configuração de Storage**
```tf
storage_type            = "nfs"
volume_size             = "300"  # GB
volume_storage_template = "SSP_Cluster_1 base template"
```

---

# **12. Considerações Finais**
- Podem ocorrer alterações e ajustes durante a execução do Terraform.
- O GitHub oficial deve ser usado como referência contínua.
- O procedimento foi validado em ambiente Power10 com PowerVC.

---
**Fim do Documento**

