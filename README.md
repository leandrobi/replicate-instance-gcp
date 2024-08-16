
# Instruções para Replicação de VMs no Google Cloud


## 1. Como Criar uma Imagem a Partir de uma VM que Já Existe

**Parar a VM** (opcional, para garantir consistência):

```bash
gcloud compute instances stop <nome-da-vm> --zone=<zona-da-vm>
```

**Criar a imagem do disco da VM:**

```bash
gcloud compute images create <nome-da-imagem> \
    --source-disk=<nome-do-disco> \
    --source-disk-zone=<zona-do-disco> \
    --project=<seu-id-do-projeto>
```

> **<nome-da-imagem>**: Nome que você escolhe para a imagem.  
> **<nome-do-disco>**: Nome do disco da VM atual (pode ser encontrado nos detalhes da VM).  
> **<zona-do-disco>**: Zona onde o disco está localizado (mesma zona da VM).  
> **<seu-id-do-projeto>**: ID do projeto onde a imagem será criada.

## 2. Como Gerar um Arquivo JSON da Configuração de Hardware da Máquina Atual

**Obter as configurações da VM e salvar em um arquivo JSON:**

```bash
gcloud compute instances describe <nome-da-vm> --zone=<zona-da-vm> --format=json > vm-config.json
```

> **<nome-da-vm>**: Nome da VM atual.  
> **<zona-da-vm>**: Zona onde a VM está localizada.

## 3. Como Criar um Script `.sh` Já Vinculado ao JSON para Recriar a VM com Parâmetros Usando `vi`

**Criar o script usando `vi`:**

```bash
vi recreate_vm.sh
```

**Inserir o seguinte conteúdo no script:**

```bash
#!/bin/bash

# Verificar se todos os parâmetros necessários foram fornecidos
if [ "$#" -lt 3 ]; then
    echo "Uso: $0 <nome-da-nova-vm> <seu-id-do-projeto> <nome-da-imagem> [disk-type]"
    exit 1
fi

# Capturar os parâmetros passados
VM_NAME=$1
PROJECT_ID=$2
IMAGE_NAME=$3
DISK_TYPE=$4  # Parâmetro opcional

# Se o DISK_TYPE não for fornecido, use o do JSON ou defina como pd-standard
if [ -z "$DISK_TYPE" ]; then
    DISK_TYPE=$(jq -r '.disks[0].type | split("/")[-1]' vm-config.json)

    # Verifica se o DISK_TYPE é válido, caso contrário, define como pd-standard
    if [[ "$DISK_TYPE" != "pd-standard" && "$DISK_TYPE" != "pd-ssd" && "$DISK_TYPE" != "pd-balanced" ]]; then
        DISK_TYPE="pd-standard"
    fi
fi

# Extraindo outras informações do JSON
ZONE=$(jq -r '.zone | split("/")[-1]' vm-config.json)
MACHINE_TYPE=$(jq -r '.machineType | split("/")[-1]' vm-config.json)
NETWORK_INTERFACE=$(jq -r '.networkInterfaces[0].network | split("/")[-1]' vm-config.json)
SUBNET=$(jq -r '.networkInterfaces[0].subnetwork | split("/")[-1]' vm-config.json)
TAGS=$(jq -r '.tags.items | join(",")' vm-config.json)
SCOPES=$(jq -r '.serviceAccounts[0].scopes | join(",")' vm-config.json)
DISK_NAME=$(jq -r '.disks[0].deviceName' vm-config.json)
DISK_SIZE=$(jq -r '.disks[0].diskSizeGb' vm-config.json)

# Comando para recriar a VM
gcloud compute instances create $VM_NAME \
    --zone=$ZONE \
    --image=$IMAGE_NAME \
    --image-project=$PROJECT_ID \
    --machine-type=$MACHINE_TYPE \
    --network-interface=network=$NETWORK_INTERFACE,subnet=$SUBNET \
    --tags=$TAGS \
    --scopes=$SCOPES \
    --boot-disk-size=${DISK_SIZE}GB \
    --boot-disk-type=$DISK_TYPE \
    --boot-disk-device-name=$DISK_NAME

echo "VM $VM_NAME recriada com sucesso na zona $ZONE."
```

**Salvar e sair do editor `vi`:**

Pressione Esc, digite `:wq` e pressione Enter.

**Tornar o script executável:**

```bash
chmod +x recreate_vm.sh
```

## Dicas

**Como listar as imagens de disco do projeto atual:**

```bash
gcloud compute images list --project=<seu-id-do-projeto>
```

> **<seu-id-do-projeto>**: ID do seu projeto no Google Cloud.

---

**Como executar o script `.sh` passando os parâmetros solicitados:**

```bash
./recreate_vm.sh <nome-da-nova-vm> <seu-id-do-projeto> <nome-da-imagem> [disk-type]
```

> **<nome-da-nova-vm>**: Nome que você quer dar para a nova VM.  
> **<seu-id-do-projeto>**: ID do seu projeto no Google Cloud.  
> **<nome-da-imagem>**: Nome da imagem que você criou a partir da VM original.  
> **[disk-type]**: (Opcional) Tipo de disco a ser usado (`pd-standard`, `pd-ssd`, `pd-balanced`). Se não for informado, será utilizado o tipo de disco do JSON ou "pd-standard".

---

**Como filtrar as imagens de um projeto específico:**

```bash
gcloud compute images list --project=<seu-id-do-projeto>
```

> Isso exibirá apenas as imagens disponíveis no projeto especificado.
