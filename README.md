# A Modern Object, Block and File Storage Unified System

**Autor:** Rafael Francisco do Prado  
**Contato:** rafael.prado12347@gmail.com  
**Instituição:** Universidade Estadual de Campinas – Unicamp  
**Unidade:** Faculdade de Tecnologia – FT  
**Curso:** Pós-Graduação Lato Sensu / Extensão em Engenharia e Administração de Sistemas de Banco de Dados  

---

## 📄 Resumo / Abstract

### Resumo
O crescimento contínuo do volume de dados exige soluções de armazenamento escaláveis e tolerantes a falhas, mas os *appliances* proprietários e os arranjos RAID locais impõem custos elevados e pontos únicos de falha. Este trabalho tem como objetivo demonstrar a viabilidade técnica do **Ceph**, plataforma *open source* de *Software-Defined Storage* (SDS) que unifica o armazenamento em blocos, objetos e arquivos, como alternativa de baixo custo aos modelos tradicionais. A proposta consistiu em apresentar a fundamentação teórica de RAID e da arquitetura do Ceph e em implantar, com **Oracle VM VirtualBox**, **Vagrant** e **`cephadm`**, um cluster de três nós sobre **Rocky Linux 9**, com seis OSDs e *pools* replicados, acompanhado de uma máquina cliente (além de uma VM isolada com RAID por software via `mdadm` para comparação). Foram configurados e validados os serviços **RBD**, **RGW** (compatível com S3) e **CephFS**, além do monitoramento pelo **Ceph Dashboard**, confrontando os resultados com o arranjo RAID tradicional. Conclui-se que o Ceph é uma solução robusta, adaptável e de bom custo-benefício, passível de implantação inclusive com o reaproveitamento de hardware.

**Palavras-chave:** Ceph; Software-Defined Storage; Armazenamento de dados.

### Abstract
*The continuous growth of data volumes demands scalable and fault-tolerant storage solutions, yet proprietary appliances and local RAID arrays impose high costs and single points of failure. This study aims to demonstrate the technical feasibility of Ceph, an open-source Software-Defined Storage (SDS) platform that unifies block, object and file storage, as a low-cost alternative to traditional models. The proposal consisted of presenting the theoretical background on RAID and on the Ceph architecture and deploying, with Oracle VM VirtualBox, Vagrant and cephadm, a three-node cluster running Rocky Linux 9, with six OSDs and replicated pools, together with a client machine. The RBD, RGW (S3-compatible) and CephFS services were configured and validated, as well as monitoring through the Ceph Dashboard, and the outcome was compared with a software RAID array. It is concluded that Ceph is a robust, adaptable and cost-effective solution that can be deployed even with reused hardware.*

**Keywords:** Ceph; Software-Defined Storage; Storage.

---

## 📐 Topologia do Ambiente de Testes

O laboratório foi implementado sobre o hipervisor **Oracle VM VirtualBox (v7.2.22)** com máquinas virtuais executando **Rocky Linux 9 (9.8 Blue Onyx)**. A topologia reúne **3 nós de cluster** e **1 máquina cliente** provisionados via **Vagrant** na rede pública (`192.168.56.0/24`), além da rede dedicada de cluster (`192.168.70.0/24`) para replicação entre OSDs e *heartbeat*. Uma quinta máquina virtual (`linux-raid`) permanece fora do cluster Ceph para validação comparativa de RAID por software (`mdadm`).

```text
====================================================================================================
                     REDE PÚBLICA — 192.168.56.0/24 (Acesso dos clientes e tráfego geral)
====================================================================================================
          │                       │                       │                       │
   (192.168.56.10)         (192.168.56.11)         (192.168.56.12)         (192.168.56.20)
          │                       │                       │                       │
 ┌────────┴────────┐     ┌────────┴────────┐     ┌────────┴────────┐     ┌────────┴────────┐
 │   ceph-node01   │     │   ceph-node02   │     │   ceph-node03   │     │   ceph-client   │
 │                 │     │                 │     │                 │     │                 │
 │ • MON           │     │ • MON           │     │ • MON           │     │ • RBD (rbd-nbd) │
 │ • MGR (ativo)   │     │ • MGR           │     │ • MDS (standby) │     │ • S3 (s3cmd)    │
 │ • MDS (ativo)   │     │ • RGW :8000     │     │ • RGW :8000     │     │ • CephFS (fuse) │
 │ • Dashboard:8443│     │                 │     │                 │     │                 │
 │ • rótulo _admin │     │                 │     │                 │     │                 │
 │                 │     │                 │     │                 │     │                 │
 │ [2x Discos 10GB]│     │ [2x Discos 10GB]│     │ [2x Discos 10GB]│     └─────────────────┘
 │  osd.1  | osd.5 │     │  osd.0  | osd.3 │     │  osd.2  | osd.4 │
 │  (sdb)  | (sdc) │     │  (sdb)  | (sdc) │     │  (sdb)  | (sdc) │     ┌ ─ ─ ─ ─ ─ ─ ─ ─ ┐
 └────────┬────────┘     └────────┬────────┘     └────────┬────────┘     │   linux-raid    │
          │                       │                       │               (Fora do cluster) 
======================================================================   │ • RAID 5 mdadm  │
    REDE DO CLUSTER — 192.168.70.0/24 (Replicação entre OSDs/Heartbeat)  │  (4x discos)    │
======================================================================   └ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

### Especificação das Máquinas Virtuais e Serviços

| Máquina | IP (Rede Pública) | Rede Cluster | Recursos (vCPU / RAM) | Discos Adicionais | Serviços / Daemons Ceph |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`ceph-node01`** | `192.168.56.10` | `192.168.70.0/24` | 2 vCPUs / 4096 MB | 2x 10 GB (`/dev/sdb`, `/dev/sdc`) | Bootstrap, `MON`, `MGR` (ativo), `MDS` (ativo), `Dashboard` (`:8443`), rótulo `_admin`, `osd.1`, `osd.5` |
| **`ceph-node02`** | `192.168.56.11` | `192.168.70.0/24` | 2 vCPUs / 3072 MB | 2x 10 GB (`/dev/sdb`, `/dev/sdc`) | `MON`, `MGR`, `RGW` (`:8000`), `osd.0`, `osd.3` |
| **`ceph-node03`** | `192.168.56.12` | `192.168.70.0/24` | 2 vCPUs / 3072 MB | 2x 10 GB (`/dev/sdb`, `/dev/sdc`) | `MON`, `MDS` (standby), `RGW` (`:8000`), `osd.2`, `osd.4` |
| **`ceph-client`** | `192.168.56.20` | N/A | 1 vCPU / 2048 MB | N/A | Cliente RBD (`rbd-nbd`), Cliente S3 (`s3cmd`), Cliente CephFS (`ceph-fuse`) |
| **`linux-raid`** | N/A (isolada) | N/A | 2 vCPUs / 3072 MB | 4x discos virtuais | Ambiente comparativo fora do cluster (RAID 5 por software via `mdadm`) |

---

## 📚 Estrutura e Síntese do Trabalho

### Seção 1: Introdução
- **Contextualização:** O volume global de dados (*Global Datasphere*) apresenta projeção de crescimento anual composto de cerca de 61% (de 33 ZB em 2018 para 175 ZB em 2025), exigindo infraestruturas escaláveis sem elevação proporcional de custos.
- **Motivação:** Superação das limitações de equipamentos dedicados (NAS/SAN) e arranjos RAID locais, nos quais falhas na controladora, no servidor ou em múltiplos discos simultâneos geram pontos únicos de falha.
- **Justificativa e Objetivo:** Viabilizar demandas como *Data Lakes*, *Data Marts*, BI e IA por meio do paradigma *Software-Defined Storage* (SDS) e do reaproveitamento de hardware de propósito geral (*commodity hardware*), reduzindo o *Total Cost of Ownership* (TCO).

### Seção 2: Fundamentação Teórica
- **Armazenamento Redundante com RAID:** Estudo dos níveis clássicos e aninhados (RAID 0, 1, 5, 6, 10, 01 e 50), métricas de confiabilidade (MTBF), diferenças entre RAID por hardware e por software (`mdadm`) e suas restrições de escala vertical.
- **Arquitetura do Ceph:**
  - **RADOS (*Reliable Autonomic Distributed Object Store*):** Camada base distribuída que garante consistência, replicação, detecção de falhas e rebalanceamento automático.
  - **Algoritmo CRUSH (*Controlled Replication Under Scalable Hashing*):** Cálculo determinístico e descentralizado executado pelos clientes e OSDs para alocar dados respeitando domínios de falha (*failure zones*), eliminando tabelas centrais de metadados.
  - **Componentes:** `MON` (Monitores e consenso), `OSD` (Daemons de armazenamento por disco), `MGR` (Gerenciamento e Dashboard), `MDS` (Metadados exclusivos do CephFS) e `librados`.

### Seção 3: Métodos
- **Ambiente Host:** Lenovo ThinkCentre M70s Gen 5 (Intel Core i5-14500, 32 GB RAM, SSD 1 TB, Windows 11 Pro) com VirtualBox 7.2.22 e Vagrant.
- **Stack Utilizada:** `cephadm` 21.2.0 (*Umbrella*), contêineres Podman executando Ceph 22.0.0 (*Vampire - dev*), pacotes clientes `ceph-common`, `rbd-nbd`, `ceph-fuse` e `s3cmd`, além de `mdadm` na VM de comparação.
- **Indicadores:** Estado de saúde do cluster (`HEALTH_OK`), quórum de monitores, OSDs `up`/`in`, validação funcional de RBD, RGW e CephFS, e cálculo de eficiência de armazenamento:
  $$E = \left(\frac{C_{\text{útil}}}{C_{\text{bruta}}}\right) \times 100$$

### Seção 4: Desenvolvimento e Resultados
- **Provisionamento e Bootstrap:** Automação de 4 VMs via Vagrant, configuração de `/etc/hosts` e chaves SSH `root` sem senha, seguida de bootstrap com `--allow-mismatched-release` e criação automática de 6 OSDs (totalizando ~60 GiB brutos).
- **Validação dos Serviços Unificados:**
  - **RBD (Blocos):** Criação do `rbd_pool` (64 PGs) e da imagem `vol-block01` (10 GiB / 2.560 objetos de 4 MiB), mapeada via `rbd-nbd` em `/dev/nbd0`, formatada em `ext4` e montada em `/mnt/rbd_volume`.
  - **RGW (Objetos):** Publicação de 2 daemons RGW (`ceph-node02` e `ceph-node03`) na porta `8000`, criação do usuário `cliente-s3`, criação do bucket `s3://bucket-demonstracao` e upload/listagem via `s3cmd`.
  - **CephFS (Arquivos):** Implantação de 2 daemons MDS (1 ativo em `ceph-node01` e 1 *standby* em `ceph-node03`), criação dos pools `cephfs_data` e `cephfs_metadata` e montagem via `ceph-fuse` em `/mnt/cephfs_share` (19 GB disponíveis).
- **Monitoramento e Comparação:** Validação via CLI (`ceph -s`, `ceph osd tree`) e **Ceph Dashboard** (`https://192.168.56.10:8443`, estado *Saudável*, 0 alertas), além da criação do arranjo RAID 5 (`/dev/md0`) com 4 discos na VM `linux-raid`.

### Seção 5 e 6: Discussão e Conclusão
- **Eficiência vs. Resiliência:** Com fator de replicação 3, os 60 GB brutos do cluster resultaram em ~19 GB úteis (~33% de eficiência), priorizando tolerância a falhas de nós inteiros; em contrapartida, o RAID 5 com 4 discos entrega 75% de eficiência, porém tolera apenas 1 disco defeituoso e mantém o servidor/controladora como ponto único de falha.
- **Lições Aprendidas e Trabalhos Futuros:** Documentação das tratativas de divergência de release no `cephadm`, permissões SSH `root` e ajuste de endpoint/porta (`8000`) no `~/.s3cfg`, apontando como próximos passos testes de falha induzida, uso de releases estáveis em hardware físico e *erasure coding*.

---

## 🛠️ Como Executar o Laboratório

### Pré-requisitos
- **Oracle VM VirtualBox** (v6.1+ / testado na v7.2.22)
- **Vagrant** (v2.2+)
- **Host com pelo menos 16 GB de RAM livres** (o cluster + cliente consomem 12 GB de RAM alocados)

### Passo a Passo Resumido

1. **Clone o repositório e suba as VMs:**
   ```bash
   git clone [https://github.com/rfprado/TCC-FT-Unicamp.git](https://github.com/rfprado/TCC-FT-Unicamp.git)
   cd TCC-FT-Unicamp
   vagrant up
   ```

2. **Acesse o nó principal (`ceph-node01`) e configure o acesso SSH sem senha entre os nós:**
   ```bash
   vagrant ssh ceph-node01
   sudo su -
   # Habilitar PermitRootLogin yes em /etc/ssh/sshd_config, reiniciar sshd e distribuir chaves SSH (ssh-copy-id)
   ```

3. **Instale o `cephadm` e execute o bootstrap do cluster:**
   ```bash
   dnf install -y cephadm
   mkdir -p /etc/ceph
   cephadm bootstrap --mon-ip 192.168.56.10 --cluster-network 192.168.70.0/24 --allow-mismatched-release
   ```

4. **Adicione os hosts (`ceph-node02` e `ceph-node03`), aplique os rótulos e provisione os OSDs:**
   ```bash
   cephadm shell
   ceph orch host add ceph-node02 192.168.56.11
   ceph orch host add ceph-node03 192.168.56.12
   ceph orch apply osd --all-available-devices
   ceph -s
   ```

5. **Acesse o Ceph Dashboard:**
   - **URL:** `https://192.168.56.10:8443`
   - **Usuário:** `admin` *(senha gerada na saída do comando `bootstrap`)*

---
*Trabalho de Conclusão de Curso apresentado à Faculdade de Tecnologia da Universidade Estadual de Campinas (FT-UNICAMP) — 2026.*
