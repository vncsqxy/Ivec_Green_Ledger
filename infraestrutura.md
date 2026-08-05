# Infraestrutura do Sistema Green Ledger

## 1. Informações do Projeto

| Campo | Descrição |
| :--- | :--- |
| **Nome do Projeto** | Green Ledger – Sistema de Rastreamento Inteligente |
| **Objetivo** | Automatizar a triagem de materiais da Iveco com cálculo automático da pegada de carbono, utilizando persistência centralizada em nuvem e integração contínua com APIs externas, garantindo rastreabilidade total |
| **Público‑alvo** | Operadores logísticos, analistas de logística e gestores de sustentabilidade da Iveco. |

## 2. Requisitos de Hardware

### 2.1 Estação de Trabalho (Cliente WPF)

| Componente | Mínimo | Recomendado |
| :--- | :--- | :--- |
| **Processador** | Intel Core i3 | Intel Core i5 ou superior |
| **Memória RAM** | 8 GB | 16 GB |
| **Armazenamento** | 600 MB livres | 1,5 GB livres |
| **Sistema Operacional** | Windows (64‑bit - versões suportadas) | Windows (64-bit - versões mais recentes) |
| **Rede** | Placa de rede Ethernet/Wi‑Fi com acesso à internet | Conexão banda larga estável |

### 2.2 Servidor (API e Nuvem)

- **API REST:** Hospedagem em servidor Windows/Linux com suporte a ASP.NET Core (pode ser máquina virtual ou contêiner).
- **Banco de Dados Cloud:** Firebase Firestore (Google Cloud Platform) – Plano Spark (gratuito) para uso inicial, escalável para Plano mais robusto sob demanda.
- **Conectividade:** Acesso HTTPS outbound liberado para APIs externas (BrasilAPI, NHTSA, Mercado Livre).

## 3. Requisitos de Software

### 3.1 Cliente Desktop

| Software | Finalidade |
| :--- | :--- |
| **Microsoft .NET Runtime** | Obrigatório para execução do aplicativo WPF |
| **Microsoft Visual C++ Redistributable** | Necessário para execução de dependências e bibliotecas nativas (ex: renderização gráfica) |
| **Windows** | Sistema operacional base (64-bit) |

### 3.2 Servidor

| Software | Finalidade |
| :--- | :--- |
| **ASP.NET Core Runtime** | Hospedar a API REST |
| **Firebase Admin SDK** | Comunicação com Firestore |
| **Certificado SSL/TLS** | Comunicação HTTPS obrigatória |

## 4. Dependências

### 4.1 Pacotes NuGet (WPF)

| Pacote | Finalidade |
| :--- | :--- |
| `Microsoft.Extensions.Http` | Cliente HTTP para consumo da API REST e serviços externos |
| `Newtonsoft.Json` | Serialização/desserialização JSON (fallback e cache local) |
| `LiveChartsCore.SkiaSharpView.WPF` | Exibição de gráficos e dashboards interativos |
| `QuestPDF` | Geração de relatórios e etiquetas em PDF |
| `Serilog.Sinks.File` | Registro de logs em arquivo local (offline e diagnóstico) |

### 4.2 Pacotes NuGet (API)

| Pacote | Finalidade |
| :--- | :--- |
| `FirebaseAdmin` | SDK oficial para autenticação e administração do Firebase |
| `Google.Cloud.Firestore` | Cliente nativo para operações CRUD no Firestore (modo Datastore) |
| `Swashbuckle.AspNetCore` | Geração automática da documentação Swagger/OpenAPI |
| `Serilog.AspNetCore` | Log estruturado no lado do servidor com sinks para arquivo e console |

> **Observação:** Todos os pacotes e runtimes acima devem ser mantidos em suas versões LTS (Long Term Support) mais recentes e estáveis, garantindo a compatibilidade entre si e com o ecossistema .NET em uso.

### 4.3 Serviços Externos (APIs de Terceiros)

| API | Função | Dependência de Internet |
| :--- | :--- | :---: |
| **BrasilAPI** | Validação de CNPJ de fornecedores | Sim |
| **NHTSA VPIC** | Decodificação do chassi (VIN) e validação da Iveco | Sim |
| **Mercado Livre Developers** | Rastreamento da rota de entrega (status) | Sim |
| **Firebase Firestore** | Persistência definitiva e dashboards | Sim (para sync) |

## 5. Arquitetura do Sistema

| Componente | Detalhes |
| :--- | :--- |
| **Cliente WPF (.NET)** | Arquitetura MVVM, logs NoSQL e sincronização com retry. |
| **API REST** | Centraliza as regras de negócio, orquestração e validações. Consome APIs externas (NHTSA, BrasilAPI, ML) e persiste os dados finais no Firebase. |
| **Banco de Dados** | Combina **SQLite** com **Firebase Firestore / NoSQL**. O Firestore armazena os dados consolidados. |

## 6. Riscos Identificados

| Risco | Probabilidade | Impacto | Mitigação |
| :--- | :--- | :--- | :--- |
| **Queda de conectividade** | Alta | Médio | Operação offline com SQLite; sincronização automática ao reconectar. |
| **Cota gratuita do Firebase excedida** | Média | Alto | Migrar para plano Blaze ou banco relacional próprio. |
| **Indisponibilidade das APIs externas** (NHTSA, BrasilAPI) | Média | Médio | Cache local de CNPJ/VIN já validados; fallback para verificação manual. |
| **Incompatibilidade de versão do .NET Runtime** | Baixa | Alto | Verificar instalação do runtime adequado no checklist de implantação; distribuir instalador junto. |
| **Falha de hardware no terminal de operação** | Baixa | Médio | Manter máquina reserva configurada; utilizar dispositivos móveis provisórios. |
| **Ataques de brut force ou SQL Injection** | Baixa | Crítico | HTTPS mandatório, CORS restrito, hash de senhas, validação de inputs. |

## 7. Plano de Contingência

### 7.1 Perda de Conexão com a Internet

- O sistema **continua operando** localmente: todos os registros de triagem são persistidos no SQLite.
- Ao restabelecer a rede, um processo em background (worker) varre o SQLite e envia os dados em lote para a API → Firestore.
- Dashboards gerenciais serão atualizados automaticamente após a sincronização.

### 7.2 Indisponibilidade de API Externa (ex.: BrasilAPI fora do ar)

- Se a validação de CNPJ falhar por timeout, o sistema exibe um alerta e permite a **digitação manual** do fornecedor, marcando o registro como “pendente de verificação fiscal”.
- Uma rotina posterior (ou job agendado) reprocessa as pendências.

### 7.3 Falha Física do Terminal de Operação

- O local de operação deve contar com um aparelho secundário ou maquinas corporativas pré-configuradas.
- Em caso de falha da máquina principal, os operadores assumem imediatamente o dispositivo reserva, que sincroniza os dados mais recentes diretamente da nuvem, minimizando o tempo de inatividade e eliminando a dependência de papel.
- O acionamento da equipe de TI ocorre apenas para o recolhimento e manutenção do equipamento defeituoso, sem paralisar o fluxo logístico.

### 7.4 Estratégia de Retenção e Redundância de Dados

- **Nuvem (Firestore):** Ativação de *Point-in-Time Recovery* (PITR) no Google Cloud, permitindo a restauração granular de dados caso ocorram exclusões acidentais, além da geração de snapshots de longo prazo.
- **Banco Local (SQLite):** Configuração de espelhamento automático ou envio via rede corporativa para um servidor NAS (Network Attached Storage) local, protegendo os dados temporários contra falhas no disco rígido da estação de trabalho.
- **Auditoria e Logs:** Todos os registros locais gerados pelo sistema são rotacionados diariamente e enviados de forma assíncrona para um bucket de armazenamento seguro na nuvem, garantindo a rastreabilidade completa em caso de sinistro.

---

*Documento elaborado para o Projeto de TCC – SENAI Nova Lima, conforme atividade prática de Infraestrutura de Software.*
*Última atualização: 04/08/2026.*
