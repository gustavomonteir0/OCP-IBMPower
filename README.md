# OCP-IBMPower

📘 GUIA COMPLETO – Deploy de Cluster OpenShift (OCP) em IBM Power via Terraform

Este documento organiza o processo de instalação do OpenShift Container Platform (OCP) em ambiente IBM Power utilizando PowerVC, Terraform e imagens RHEL + RHCOS.

🧩 1. Pré-requisitos
✔️ Softwares e arquivos necessários

ISO do RHEL 8.8

Imagem CoreOS (RHCOS) compatível com Power (ppc64le)

Acesso à HMC, PowerVC, VS Code e ao repositório GitHub:

https://github.com/ocp-power-automation/ocp4-upi-powervm

🖥️ 2. Criar LPAR RHEL 8.8 para Bastion Host
Configuração da LPAR (via HMC)

4 vCPUs

16 GB RAM

100 GB disco

Procedimento

Instale o RHEL 8.8.

Registre o sistema na Red Hat.

Atualize até 8.10:

sudo yum update -y
sudo yum upgrade -y


Baixe a ISO do CoreOS para dentro da máquina.

Crie o usuário core e gere chaves SSH:

ssh-keygen


Copie as chaves para o local onde ficará o script:

ocp4-upi-powervm-main/data

🧰 3. Instalar cloud-init no RHEL
sudo dnf install cloud-init -y

🔧 4. Instalar RMC (DynamicRM)
wget https://public.dhe.ibm.com/software/server/POWER/Linux/yum/download/ibm-power-repo-latest.noarch.rpm
rpm -ivh --nodeps ibm-power-repo-latest.noarch.rpm
cd /opt/ibm/lop
./configure 
yum search rm
yum install DynamicRM -y


Após isso, importe a LPAR no PowerVC, onde ela aparecerá em VM List.

📀 5. Criar Imagem do Bastion no PowerVC

Em Image List, criar imagem baseada na VM RHEL pronta.

Em Available Volumes, anexar:

disco boot original

disco adicional de 120 GB

Essa VM será usada para gerar a imagem CoreOS.

🛠️ 6. Preparar a Imagem RHCOS no disco 120GB

Acesse via HMC e instale pacotes:

yum install wget qemu-img parted gzip -y


A imagem ISO deve estar previamente no sistema.

Descompactar imagem CoreOS
gunzip rhcos-openstack.ppc64le.qcow2.gz

Converter QCOW2 para RAW
qemu-img convert -f qcow2 -O raw rhcos-openstack.ppc64le.qcow2 rhcos-latest.raw

Localizar o disco vazio (120 GB)
lsblk

Gravar imagem RAW no disco
dd if=rhcos-latest.raw of=${empty_disk_device} bs=4M


${empty_disk_device} = caminho do disco, ex: /dev/sdb

📦 7. Criar Imagem CoreOS no PowerVC

Desanexar o volume da VM.

PowerVC → Images → Create Image

Nome: cores-img

Hypervisor: PowerVM

OS: COREOS

Endianness: littleEndian

Adicionar volume modificado (imagem RHCOS)

Definir Boot = Yes

Criar imagem.

Agora você possui:

Imagem RHEL → Bastion

Imagem RHCOS → Masters/Workers

🧬 8. Criar Compute Templates no PowerVC

Templates recomendados:

Tipo	vCPUs	RAM	Disco
Bootstrap	2	16 GB	120 GB
Master	2	32 GB	120 GB
Worker	2	32 GB	120 GB
Bastion	2	16 GB	200 GB

Importante: nomes devem ser idênticos ao Terraform, como:

ocp.bastion

ocp.bootstrap

ocp.master

ocp.worker

🌐 9. Configurar Rede no PowerVC

Nome da rede deve ser o mesmo usado no Terraform.

Modo estático.

O DHCP será controlado pelo Bastion Host.

Configurar o DHCP para ignorar MACs das LPARs do cluster.

🧱 10. Configurar Terraform

Repositório GitHub:

https://github.com/ocp-power-automation/ocp4-upi-powervm


O arquivo principal editado será:

var.tfvars

Executar Terraform
terraform init
terraform apply -var-file var.tfvars

🏗️ 11. Exemplo de Configurações (var.tfvars)
PowerVC Details
auth_url                    = "https://192.168.140.115:5000/v3/"
user_name                   = "root"
password                    = "fit123"
tenant_name                 = "ibm-default"
domain_name                 = "Default"
openstack_availability_zone = "Default Group:902821B_78B51B1"

Rede
network_name = "openshift-p10"


Copiar o image ID correto no PowerVC → Image List → MTMS.

OpenShift Cluster Details
bastion = {
  instance_type = "ocp.bastion",
  image_id      = "ba1f2164-5c90-4be2-a23e-c5ed83a67d51",
  count         = 1,
  fixed_ip_v4   = "192.168.140.150"
}

bootstrap = {
  instance_type = "ocp.bootstrap",
  image_id      = "e0f7fe48-c2fe-4a58-904b-7c00b82aad43",
  count         = 0,
  fixed_ips     = ["192.168.140.151"]
}

master = {
  instance_type = "ocp.master",
  image_id      = "e0f7fe48-c2fe-4a58-904b-7c00b82aad43",
  count         = 3,
  fixed_ips     = [
    "192.168.140.152",
    "192.168.140.153",
    "192.168.140.154"
  ]
}

worker = {
  instance_type = "ocp.worker",
  image_id      = "e0f7fe48-c2fe-4a58-904b-7c00b82aad43",
  count         = 2,
  fixed_ips     = [
    "192.168.140.155",
    "192.168.140.156"
  ]
}

Storage
storage_type                = "nfs"
volume_size                 = "300"
volume_storage_template     = "SSP_Cluster_1 base template"

⚠️ 12. Observações Finais

Pequenos ajustes podem surgir dependendo do ambiente.

O GitHub oficial contém instruções complementares.

Recomenda-se utilizar VS Code com extensão Terraform instalada.
