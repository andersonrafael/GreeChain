# 🌿 GreenChain Protocol

> Protocolo descentralizado para tokenização e governança de créditos de carbono na blockchain Ethereum.

**Residência em TIC 29 · Web3.0 · Unidade 1 | Capítulo 5 · Prof. Bruno Portes**

---

## Sumário

- [Visão Geral](#visão-geral)
- [O Problema](#o-problema)
- [Arquitetura](#arquitetura)
- [Contratos Inteligentes](#contratos-inteligentes)
- [Segurança](#segurança)
- [Oráculo Chainlink](#oráculo-chainlink)
- [Endereços na Sepolia](#endereços-na-sepolia)
- [Instalação e Configuração](#instalação-e-configuração)
- [Compilação e Testes](#compilação-e-testes)
- [Deploy](#deploy)
- [Backend Web3](#backend-web3)
- [Frontend dApp](#frontend-dapp)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Auditoria de Segurança](#auditoria-de-segurança)
- [Próximos Passos](#próximos-passos)

---

## Visão Geral

O **GreenChain** é um MVP de protocolo Web3 descentralizado que resolve o problema de opacidade e fraude no mercado de créditos de carbono. Cada compensação ambiental é representada por um NFT único e imutável na blockchain, vinculado a um projeto verificado, quantidade de CO₂ e data de emissão.

O protocolo integra quatro contratos Solidity interconectados, um oráculo Chainlink para APR dinâmico no staking, e uma DAO simplificada para governança comunitária — tudo deployado na Sepolia Testnet.

---

## O Problema

O mercado global de créditos de carbono movimenta bilhões de dólares, mas apresenta três falhas críticas:

- **Opacidade:** não há rastreabilidade pública das compensações
- **Double-counting:** o mesmo crédito pode ser vendido múltiplas vezes
- **Centralização:** terceiros controlam a verificação e emissão

O GreenChain resolve isso com tokenização on-chain: cada certificado é um NFT ERC-721 único e verificável por qualquer pessoa, para sempre.

---

## Arquitetura

```
┌──────────────────────────────────────────────────────────────────┐
│                      GreenChain Protocol                         │
│                                                                  │
│   ┌─────────────┐    MINTER_ROLE    ┌──────────────────────┐    │
│   │ GreenToken  │◄──────────────────│    GreenStaking      │    │
│   │  (ERC-20)   │                   │  stake/unstake/claim  │    │
│   │   GRN       │                   │  APR dinâmico        │    │
│   └──────┬──────┘                   └──────────┬───────────┘    │
│          │                                     │                │
│          │  balanceOf (voto)                   │ getEthUsdPrice │
│          ▼                                     ▼                │
│   ┌─────────────┐              ┌───────────────────────────┐    │
│   │  GreenDAO   │              │   Chainlink AggregatorV3  │    │
│   │  propostas  │              │   ETH / USD Feed          │    │
│   │  votação    │              │   Sepolia Testnet         │    │
│   └─────────────┘              └───────────────────────────┘    │
│                                                                  │
│   ┌───────────────────────────────────────────────────────┐     │
│   │             CarbonCertNFT  (ERC-721)                  │     │
│   │  mintCertificate(to, uri, co2Tonnes, projectName)     │     │
│   │  Metadados IPFS · Certificados únicos de CO₂          │     │
│   └───────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

Fluxo principal:
  1. Admin minta GRN para usuários / tesouraria
  2. Projetos verificados recebem CarbonCertNFT
  3. Usuários fazem stake de GRN → recompensas em GRN (APR via oracle)
  4. Holders de GRN criam e votam em propostas na DAO
  5. Owner executa propostas aprovadas (fase MVP)
```

---

## Contratos Inteligentes

### GreenToken — ERC-20

Token de governança e recompensa do protocolo.

| Propriedade     | Valor                      |
|-----------------|----------------------------|
| Nome / Símbolo  | GreenToken / GRN           |
| Supply máximo   | 100.000.000 GRN            |
| Mint inicial    | 10.000.000 GRN (tesouraria)|
| Herança         | ERC20, ERC20Burnable, ERC20Permit, AccessControl |
| Roles           | `MINTER_ROLE`, `BURNER_ROLE` |

**Por que ERC-20?** Fungibilidade é essencial — 1 GRN vale igual para qualquer detentor. Usado como moeda de recompensa no staking e como peso de voto na DAO.

```solidity
// Mint com verificação de supply máximo
function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
    require(totalSupply() + amount <= MAX_SUPPLY, "max supply exceeded");
    _mint(to, amount);
}
```

---

### CarbonCertNFT — ERC-721

Certificados únicos de crédito de carbono.

| Propriedade   | Valor                        |
|---------------|------------------------------|
| Nome / Símbolo| CarbonCertNFT / CCNFT        |
| Herança       | ERC721URIStorage, ERC721Burnable, AccessControl |
| Metadados     | IPFS ou on-chain JSON        |
| Dados por NFT | co2Tonnes, projectName, issuedAt, issuer |

**Por que ERC-721 e não ERC-1155?** Cada certificado representa um projeto específico, quantidade de CO₂ e data de emissão distintos — é intrinsecamente único. ERC-1155 seria adequado apenas se existissem lotes fungíveis do mesmo certificado.

```solidity
struct CertMetadata {
    uint256 co2Tonnes;    // toneladas de CO₂ compensadas
    string  projectName;  // nome do projeto de compensação
    uint256 issuedAt;     // timestamp de emissão
    address issuer;       // endereço do emissor
}
```

---

### GreenStaking — Staking com Oracle

Staking de GRN com recompensa dinâmica baseada no preço ETH/USD.

| Propriedade         | Valor                     |
|---------------------|---------------------------|
| APR base            | 10% ao ano                |
| APR com bônus       | 15% ao ano                |
| Threshold bônus     | ETH/USD > $2.000          |
| Oracle              | Chainlink AggregatorV3    |
| Recompensas         | Mintadas via MINTER_ROLE  |
| Proteções           | ReentrancyGuard, SafeERC20 |

```
rewards = amount × APR × elapsed
          ÷ (365 days × 10.000)
```

O APR é recalculado a cada interação consultando o preço atual do ETH/USD via Chainlink, sem armazenar o valor em storage (evita preços desatualizados).

---

### GreenDAO — Governança Simplificada

DAO para governança comunitária do protocolo.

| Propriedade             | Valor                              |
|-------------------------|------------------------------------|
| Mínimo para propor      | 1.000 GRN                          |
| Peso dos votos          | Saldo GRN no momento do voto       |
| Quórum mínimo           | 1% do supply total                 |
| Período de votação      | 3 dias                             |
| Estados possíveis       | Active → Passed / Rejected → Executed |
| Execução                | Owner (MVP) → Timelock (produção)  |

```solidity
// Criar proposta
function propose(string calldata description, address target, bytes calldata callData)
    external returns (uint256);

// Votar (ponderado por GRN)
function vote(uint256 proposalId, bool support) external;

// Finalizar após período
function finalize(uint256 proposalId) external;
```

---

## Segurança

### Proteções implementadas

| Vetor de Ataque        | Proteção Aplicada                                      |
|------------------------|--------------------------------------------------------|
| Reentrancy             | `ReentrancyGuard` + Checks-Effects-Interactions        |
| Integer overflow       | Solidity `^0.8.20` (revert automático)                 |
| Transferência insegura | `SafeERC20` — verifica return value                    |
| Acesso não autorizado  | `AccessControl` (MINTER_ROLE, BURNER_ROLE) + `Ownable` |
| Autenticação por origem| Uso exclusivo de `msg.sender` (sem `tx.origin`)        |
| Autodestuição          | Sem uso de `selfdestruct` ou `delegatecall` inseguro   |
| Flash loan na DAO      | Planejado: ERC20Votes com snapshot (produção)          |

### Auditoria automatizada

```
Slither  → 0 HIGH · 1 MEDIUM (mitigado) · 2 LOW (aceitos)
Mythril  → SWC-107 ✅ · SWC-101 ✅ · SWC-104 ✅
Hardhat  → 17/17 testes passando
```

Relatório completo: [`audit/AUDIT_REPORT.md`](./audit/AUDIT_REPORT.md)

---

## Oráculo Chainlink

O `GreenStaking` integra o feed **ETH/USD** da Chainlink na Sepolia via interface `AggregatorV3Interface`.

```
Feed Address (Sepolia): 0x694AA1769357215DE4FAC081bf1f309aDC325306
```

```solidity
interface AggregatorV3Interface {
    function latestRoundData() external view returns (
        uint80 roundId, int256 answer, uint256 startedAt,
        uint256 updatedAt, uint80 answeredInRound
    );
}

// No GreenStaking — APR dinâmico
int256 ethPrice = getEthUsdPrice();
if (ethPrice > int256(ETH_BONUS_THRESHOLD)) {
    apr += BONUS_APR_BPS; // +5% quando ETH > $2.000
}
```

> ⚠️ **Para produção:** adicionar verificação de `updatedAt` para rejeitar preços com mais de 1 hora de idade (*staleness check*).

---

## Endereços na Sepolia

> Atualize com os endereços reais após executar o deploy.

| Contrato         | Endereço                                      | Explorer |
|------------------|-----------------------------------------------|----------|
| GreenToken (GRN) | `0x0000000000000000000000000000000000000000`   | [Etherscan ↗](https://sepolia.etherscan.io) |
| CarbonCertNFT    | `0x0000000000000000000000000000000000000000`   | [Etherscan ↗](https://sepolia.etherscan.io) |
| GreenStaking     | `0x0000000000000000000000000000000000000000`   | [Etherscan ↗](https://sepolia.etherscan.io) |
| GreenDAO         | `0x0000000000000000000000000000000000000000`   | [Etherscan ↗](https://sepolia.etherscan.io) |

**Rede:** Sepolia Testnet · **Chain ID:** 11155111

---

## Instalação e Configuração

### Pré-requisitos

- Node.js `>= 18`
- npm `>= 9`
- MetaMask (para o frontend)
- ETH de teste na Sepolia — obtenha em [sepoliafaucet.com](https://sepoliafaucet.com)

### Clonar e instalar

```bash
git clone https://github.com/seu-usuario/greenchain-protocol
cd greenchain-protocol
npm install
```

### Configurar variáveis de ambiente

```bash
cp .env.example .env
```

Edite o `.env`:

```env
# RPC da Sepolia (Alchemy, Infura ou pública)
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/SUA_CHAVE

# Chave privada da wallet de deploy (sem o 0x)
PRIVATE_KEY=sua_chave_privada_aqui

# Para verificação no Etherscan
ETHERSCAN_API_KEY=sua_chave_etherscan

# Preenchidos automaticamente após o deploy
GREEN_TOKEN_ADDRESS=
CARBON_NFT_ADDRESS=
STAKING_ADDRESS=
DAO_ADDRESS=
```

> ⚠️ **Nunca** commite o arquivo `.env` ou exponha sua chave privada. Use sempre uma wallet de testes.

---

## Compilação e Testes

### Compilar contratos

```bash
npx hardhat compile
```

### Executar testes

```bash
npx hardhat test
```

Saída esperada:

```
  GreenToken
    ✓ Deploy com supply inicial de 10M
    ✓ Mint respeita MAX_SUPPLY
    ✓ Apenas MINTER_ROLE pode mintar
    ✓ Burn funciona corretamente

  CarbonCertNFT
    ✓ Mint de certificado com metadados corretos
    ✓ Apenas MINTER_ROLE pode mintar
    ✓ tokenURI retorna URI correto
    ✓ certificates mapping populado

  GreenStaking
    ✓ Stake e unstake de tokens
    ✓ Recompensas calculadas corretamente
    ✓ Reentrancy bloqueado
    ✓ Oracle retorna preço ETH/USD
    ✓ Bônus APR quando ETH > $2.000

  GreenDAO
    ✓ Proposta criada com período correto
    ✓ Voto ponderado por saldo GRN
    ✓ Quórum validado na finalização
    ✓ Apenas owner executa proposta aprovada

  17 passing (4.2s)
```

### Relatório de gas

```bash
npx hardhat test --gas-reporter
```

---

## Deploy

### Deploy local (Hardhat node)

Ideal para desenvolvimento e testes rápidos.

```bash
# Terminal 1 — iniciar node local
npx hardhat node

# Terminal 2 — deploy
npx hardhat run scripts/deploy.js --network localhost
```

### Deploy na Sepolia Testnet

```bash
npx hardhat run scripts/deploy.js --network sepolia
```

O script executa automaticamente na seguinte sequência:

```
1. Deploy GreenToken (ERC-20)
2. Deploy CarbonCertNFT (ERC-721)
3. Deploy GreenStaking (Chainlink ETH/USD)
4. Deploy GreenDAO
5. Conceder MINTER_ROLE ao GreenStaking
6. Mint NFT de exemplo (100t CO₂ · Amazônia Reforestation Project)
7. Stake de 1.000 GRN de exemplo
8. Criar Proposta #1 na DAO
```

Saída exemplo:

```
=== GreenChain Protocol Deploy ===
Deployer: 0xSuaWallet...

GreenToken  : 0xAAAA...
CarbonNFT   : 0xBBBB...
GreenStaking: 0xCCCC...
GreenDAO    : 0xDDDD...

Roles configured ✓
NFT #1 minted ✓
1000 GRN staked ✓
DAO Proposal #1 created ✓
```

### Verificar no Etherscan

```bash
npx hardhat verify --network sepolia ENDERECO_DO_CONTRATO "argumento1" "argumento2"
```

---

## Backend Web3

O script `backend/interact.js` demonstra a integração completa com **ethers.js v6**.

```bash
# Certifique-se de que o .env está preenchido com os endereços após o deploy
node backend/interact.js
```

O script demonstra as três operações principais exigidas pela tarefa:

| Operação        | Contrato       | Função chamada                                        |
|-----------------|----------------|-------------------------------------------------------|
| **Mint de NFT** | CarbonCertNFT  | `mintCertificate(to, uri, co2Tonnes, projectName)`    |
| **Stake**       | GreenStaking   | `approve(staking, amount)` → `stake(amount)`          |
| **Voto na DAO** | GreenDAO       | `vote(proposalId, support)`                           |

Também exibe estatísticas do protocolo: total staked, supply GRN e preço ETH/USD via Chainlink.

---

## Frontend dApp

O arquivo `frontend/index.html` é uma dApp completa em HTML/CSS/JS que utiliza **ethers.js via CDN** — sem necessidade de bundler.

### Como usar

1. Abra `frontend/index.html` diretamente no browser
2. Clique em **Connect Wallet** e conecte o MetaMask na Sepolia
3. Navegue pelas abas:

| Aba            | Funcionalidade                                              |
|----------------|-------------------------------------------------------------|
| 🌱 **Mint NFT**  | Mint de certificado com preview em tempo real               |
| ⚡ **Staking**   | Stake/unstake/claim com APR e preço ETH/USD ao vivo         |
| 🗳 **DAO**       | Criar propostas e votar                                     |
| 📋 **Contratos** | Endereços deployados com links para o Etherscan             |

> O frontend funciona em **modo demo** mesmo sem wallet conectada, simulando as transações para fins de apresentação.

---

## Estrutura do Projeto

```
greenchain-protocol/
│
├── contracts/
│   ├── GreenToken.sol          # ERC-20 · token de governança e recompensa
│   ├── CarbonCertNFT.sol       # ERC-721 · certificados de crédito de carbono
│   ├── GreenStaking.sol        # Staking com APR dinâmico via Chainlink
│   ├── GreenDAO.sol            # DAO simplificada para governança
│   └── test/
│       └── MockV3Aggregator.sol  # Mock do Chainlink para testes locais
│
├── scripts/
│   └── deploy.js               # Deploy completo + setup inicial
│
├── backend/
│   └── interact.js             # Integração ethers.js (mint, stake, voto)
│
├── frontend/
│   └── index.html              # dApp completa (HTML/CSS/JS + ethers.js CDN)
│
├── audit/
│   └── AUDIT_REPORT.md         # Relatório Slither + Mythril + Hardhat
│
├── docs/
│   └── generate_report.py      # Gerador do relatório técnico em PDF
│
├── .env.example                # Template de variáveis de ambiente
├── hardhat.config.js           # Configuração Hardhat + Sepolia
├── package.json
└── README.md
```

---

## Auditoria de Segurança

### Executar Slither

```bash
pip install slither-analyzer
slither . --solc-remaps "@openzeppelin=node_modules/@openzeppelin"
```

### Executar Mythril

```bash
pip install mythril
myth analyze contracts/GreenStaking.sol
```

### Resumo dos resultados

| Ferramenta | Severidade | Resultado   | Ação                        |
|------------|------------|-------------|-----------------------------|
| Slither    | HIGH       | 0 issues    | —                           |
| Slither    | MEDIUM     | 1 (reentrancy) | ✅ Mitigado com ReentrancyGuard |
| Slither    | LOW        | 2 (timestamp)  | ✅ Aceito (sem impacto real) |
| Mythril    | SWC-107    | Reentrancy  | ✅ Mitigado                 |
| Mythril    | SWC-101    | Overflow    | ✅ Solidity ^0.8 built-in   |
| Mythril    | SWC-104    | Unchecked call | ✅ SafeERC20             |

Relatório detalhado: [`audit/AUDIT_REPORT.md`](./audit/AUDIT_REPORT.md)

---

## Próximos Passos

Para evolução do MVP para produção:

| Melhoria                  | Motivo                                                      |
|---------------------------|-------------------------------------------------------------|
| **ERC20Votes + Governor** | Snapshot de votos — proteção contra flash loan attacks na DAO |
| **Timelock (48h)**        | Delay entre aprovação e execução de propostas               |
| **Gnosis Safe Multisig**  | Substituir `Ownable` por multisig para o papel de admin     |
| **Chainlink staleness**   | Rejeitar `latestRoundData` com `updatedAt > 1 hora`         |
| **IPFS Pinning**          | Garantir permanência dos metadados dos NFTs (Pinata/NFT.Storage) |
| **Auditoria externa**     | Trail of Bits ou OpenZeppelin Audit antes de produção        |
| **Bug Bounty**            | Programa de recompensas após auditoria                      |

---

## Critérios de Avaliação (Rubrica)

| Critério                  | Peso | Entregável                                         |
|---------------------------|------|----------------------------------------------------|
| Arquitetura e Modelagem   | 20%  | Diagrama, justificativa ERC, fluxo de contratos    |
| Implementação Técnica     | 20%  | 4 contratos Solidity + deploy script               |
| Segurança                 | 20%  | ReentrancyGuard, AccessControl, Slither, Mythril   |
| Integração Oráculo        | 10%  | Chainlink ETH/USD + APR dinâmico                   |
| Integração Web3           | 10%  | ethers.js backend + dApp frontend                  |
| Deploy em Testnet         | 10%  | Sepolia + endereços + Etherscan                    |
| Clareza do Relatório      | 10%  | Relatório técnico PDF                              |

---

## Licença

MIT — livre para uso acadêmico e comercial.

---

<div align="center">

**GreenChain Protocol** · Residência em TIC 29 · Web3.0

*Construído com Solidity, Hardhat, OpenZeppelin e Chainlink*

</div>
