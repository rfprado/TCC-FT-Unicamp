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

O laboratório foi implementado sobre o hipervisor **Oracle VM VirtualBox (v7.2.22)** com máquinas virtuais executando **Rocky Linux 9 (9.8 Blue Onyx)**[cite: 10, 16]. A topologia reúne **3 nós de cluster** e **1 máquina cliente** provisionados via **Vagrant** na rede pública (`192.168.56.0/24`), além da rede dedicada de cluster (`192.168.70.0/24`) para replicação entre OSDs e *heartbeat*[cite: 10, 14, 15]. Uma quinta máquina virtual (`linux-raid`) permanece fora do cluster Ceph para validação comparativa de RAID por software (`mdadm`)[cite: 10, 14, 15].

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
