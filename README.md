# A Modern Object, Block and File Storage Unified System

**Autor:** Rafael Francisco do Prado  
**Contato:** rafael.prado12347@gmail.com  
**Instituição:** Universidade Estadual de Campinas – Unicamp  
**Unidade:** Faculdade de Tecnologia – FT  
**Curso:** Pós-Graduação Latu Sensu / Extensão em Engenharia e Administração de Sistemas de Banco de Dados  
**Orientador:** Prof. Dr. André F. De Angelis  

---

## 📄 Resumo / Abstract

### Resumo
Este trabalho tem como objetivo principal demonstrar as vantagens do sistema **Ceph** frente aos modelos tradicionais de armazenamento (NAS/SAN e arranjos RAID baseados em hardware). A pesquisa aborda desde os conceitos fundamentais e a arquitetura distribuída até a implantação e gerenciamento prático, evidenciando os diferenciais do modelo *Software-Defined Storage* (SDS). Através do desenvolvimento de um ambiente virtualizado de laboratório com 4 nós sob a distribuição Rocky Linux 9, são validados os mecanismos de replicação, auto-recuperação (*self-healing*) e alta disponibilidade. Conclui-se que a solução open-source Ceph apresenta elevada flexibilidade, resiliência e excelente relação custo-benefício para infraestruturas computacionais de diferentes portes.

**Palavras-chave:** Ceph; Software-Defined Storage; Armazenamento Distribuído; Algoritmo CRUSH; RADOS; Rocky Linux.

### Abstract
*This study aims to demonstrate the technical advantages of the Ceph storage system compared to traditional hardware-bound storage models. The research spans from fundamental architecture to practical deployment and cluster management, highlighting the benefits of the Software-Defined Storage (SDS) paradigm. Through a 4-node virtualized environment running Rocky Linux 9, replication mechanisms, self-healing capabilities, and high availability were validated. The findings confirm that Ceph offers a highly adaptable, robust, and cost-effective open-source solution for enterprise storage infrastructure.*

**Keywords:** Ceph; Software-Defined Storage; Distributed Storage; CRUSH Algorithm; RADOS; Rocky Linux.

---

## 📐 Topologia da Infraestrutura de Laboratório

A arquitetura do laboratório foi projetada sobre o hipervisor Oracle VM VirtualBox e automatizada via Vagrant. O cluster é composto por 4 nós virtuais operando **Rocky Linux 9**, segregados em duas redes distintas para isolamento de tráfego de aplicação e sincronismo interno.

```
                  ===================================================
                  REDE PÚBLICA / GERENCIAMENTO (192.168.56.0/24)
                  ===================================================
                          │               │               │
                 (192.168.56.10) (192.168.56.11) (192.168.56.12)  (192.168.56.20)
                          │               │               │               │
                   ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
                   │ ceph-node01 │ │ ceph-node02 │ │ ceph-node03 │ │ ceph-client │
                   │  (Bootstrap │ │ (MON, MGR,  │ │ (MON, OSD,  │ │ (RBD/CephFS │
                   │ MON, MGR,   │ │  OSD, RGW)  │ │  MDS, RGW)  │ │  Client)    │
                   │ OSD, Admin) │ │             │ │             │ │             │
                   └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └─────────────┘
                          │               │               │
                  ===================================================
                  REDE DE CLUSTER / REPLICAÇÃO (192.168.70.0/24)
                  ===================================================
```

### Especificação dos Nós do Cluster

| Nó | Hostname | IP Público (eth1) | IP Cluster (eth2) | Recursos | Daemons Ceph / Funções |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Nó 1** | `ceph-node01` | `192.168.56.10` | `192.168.70.10` | 2 vCPUs, 4 GB RAM, 2x Discos OSD (10GB) | Bootstrap, MON, MGR, OSD.0, OSD.1, CLI Admin, Dashboard |
| **Nó 2** | `ceph-node02` | `192.168.56.11` | `192.168.70.11` | 2 vCPUs, 3 GB RAM, 2x Discos OSD (10GB) | MON, MGR, OSD.2, OSD.3, RGW |
| **Nó 3** | `ceph-node03` | `192.168.56.12` | `192.168.70.12` | 2 vCPUs, 3 GB RAM, 2x Discos OSD (10GB) | MON, OSD.4, OSD.5, MDS (CephFS), RGW |
| **Cliente** | `ceph-client` | `192.168.56.20` | N/A | 1 vCPU, 2 GB RAM, 1x Disco SO (20GB) | Cliente RBD, Cliente CephFS, Cliente S3/REST API |

---

## 📚 Síntese dos Capítulos

### Capitulo 1: Introdução, Motivação e Proposta
* **Contextualização:** O volume de dados corporativos cresce a taxas anuais entre 40% e 60%, exigindo uma mudança de paradigma da infraestrutura rígida para modelos orientados a software (*Software-Defined Storage - SDS*).
* **Motivação:** Superação das limitações físicas e financeiras de storages tradicionais (NAS/SAN) e acoplamentos rígidos de hardware. O modelo tradicional apresenta o risco de ponto único de falha em controladoras e custos de licenciamento prohibitivos para projetos de inovação (Data Lakes, BI, IA).
* **Justificativa:** Demonstração da otimização de Total Cost of Ownership (TCO) na Administração Pública e em empresas privadas por meio do reuso de hardware genérico (*commodity hardware*) associado a softwares de código aberto de classe empresarial.
* **Proposta:** Implantação e análise de viabilidade de uma plataforma unificada Ceph capaz de prover simultaneamente armazenamento por bloco (RBD), sistema de arquivos distribuído (CephFS) e armazenamento de objetos (RGW S3-compatible).

### Capítulo 2: Fundamentação Teórica e Arquitetura Ceph
* **Evolução do Armazenamento Redundante:** Análise histórica e técnica do modelo RAID (níveis 0, 1, 5, 6, 10, 50) e gerenciamento por software via utilitário `mdadm` no Linux. Identificação das vulnerabilidades do RAID em cenários de degradação massiva de discos e falhas na reconstrução de arrays de grande capacidade.
* **Arquitetura Unificada Ceph:** 
  * **Algoritmo CRUSH (*Controlled Replication Under Scalable Hashing*):** Eliminação do gargalo centralizado de tabelas de metadados, substituído por um cálculo determinístico distribuído para alocação e recuperação de objetos.
  * **RADOS (*Reliable Autonomic Distributed Object Store*):** Camada base responsável pelo espelhamento, detecção de falhas e rebalanceamento automático de dados no cluster.
  * **Componentes Principais:** Monitores (MON), Servidores de Armazenamento de Objetos (OSD), Servidores de Metadados (MDS), Gerenciadores (MGR) e a biblioteca de integração `librados`.

### Capítulo 3: Metodologia e Provisionamento Prático
* **Automação de Infraestrutura:** Criação de ambiente reprodutível utilizando Vagrant e hipervisor VirtualBox.
* **Preparação de Nós (Rocky Linux 9):**
  * Configuração de resolução local (`/etc/hosts`) e chaves SSH sem senha para orquestração remota pelo `cephadm`.
  * Sincronização de tempo precisa via serviço `chronyd` (essencial para manutenção do consenso Paxos no Ceph).
  * Gerenciamento de containers via `podman` e isolamento de dependências pelo orquestrador nativo.

### Capítulo 4: Resultados, Validação e Discussão
* **Mecanismos de Resiliência:** Validação de tolerância a falhas através do desligamento abrupto de nós e desconexão de discos OSD.
* **Auto-Recuperação (*Self-Healing*):** Observação do comportamento do algoritmo CRUSH reconstruindo a paridade e as réplicas ausentes nos OSDs remanescentes sem indisponibilidade dos serviços clientes.
* **Análise de Desempenho e Viabilidade:** Comparativo econômico e funcional entre soluções proprietárias e a plataforma open-source Ceph.

---

## 🛠️ Como Executar o Laboratório

### Pré-requisitos
* Oracle VM VirtualBox (v6.1+)
* Vagrant (v2.2+)
* Mínimo de 16 GB de RAM disponível no Host

### Passo a Passo
1. Clone este repositório:
   ```bash
   git clone https://github.com/rafaelfprado/ceph-tcc-unicamp.git
   cd ceph-tcc-unicamp
   ```
2. Inicie o provisionamento das máquinas virtuais:
   ```bash
   vagrant up
   ```
3. Aceda ao nó principal (`ceph-node01`):
   ```bash
   vagrant ssh ceph-node01
   ```
4. Verifique o status da comunicação e inicialização do cluster Ceph:
   ```bash
   sudo cephadm shell -- ceph status
   ```

---
*Trabalho de Conclusão de Curso apresentado à Faculdade de Tecnologia da Universidade Estadual de Campinas (FT-UNICAMP).*
