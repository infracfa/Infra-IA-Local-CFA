# 🛡️ Guia Técnico da Fase 2: Redes, Infraestrutura e Segurança — CFA

> **Público-alvo:** Administrador de Redes, Segurança da Informação e Equipe de Infraestrutura do CFA.  
> **Servidor Alvo:** `bulma` (Oracle Linux 10.2 nativo, Intel Core i7-8700, 32GB RAM, GPU NVIDIA GTX 1060 6GB).  
> **Objetivo:** Publicar a API da IA Local do CFA de forma segura, estável e resiliente, protegendo o hardware de inferência contra sobrecargas e ataques.

---

## 🧭 Visão Geral e Arquitetura de Defesa em Profundidade

Na Fase 1, comprovamos que a IA responde com precisão milimétrica usando os documentos oficiais do CFA. No entanto, modelos de IA generativa possuem características muito diferentes de servidores web convencionais:
1. **Consumo de VRAM Fixo:** A GTX 1060 possui 6GB GDDR5. Ela processa **1 inferência profunda por vez** (`OLLAMA_NUM_PARALLEL=1`).
2. **Custo Computacional Alto:** Se 20 pessoas ou robôs fizerem perguntas no mesmo segundo sem controle, a GPU enfileira as tarefas, a memória satura e o tempo de resposta sobe de 3 segundos para minutos (ou causa instabilidade).

Por isso, aplicamos o princípio de **Defesa em Profundidade (Defense-in-Depth)**:

```
                            FLUXO DE SEGURANÇA E REDE
                            
  [ Usuário / Portal Web CFA ]
               │
               ▼ (HTTPS / 443 público)
    ┌────────────────────────────────────────────────────────┐
    │  SOPHOS FIREWALL (Perímetro Corporativo CFA)          │
    │  - WAF: Validação de requisições HTTP/JSON             │
    │  - Rate Limiting: Bloqueio de abuso por IP             │
    │  - IPS: Bloqueio de exploits, scanners e bots          │
    │  - Terminação SSL: Certificado institucional           │
    └────────────────────────────────────────────────────────┘
               │
               ▼ (Rede Interna / VLAN Restrita de Serviços)
    ┌────────────────────────────────────────────────────────┐
    │  SERVIDOR LINUX 'bulma' (Oracle Linux 10.2)           │
    │                                                        │
    │  [ firewalld / SELinux ]                               │
    │    ├── Libera TCP 8000 (ou 443 Nginx) APENAS p/ WAF   │
    │    └── Bloqueia qualquer acesso externo a 11434 e 6333│
    │                                                        │
    │  [ Nginx Proxy Reverso Local ]                         │
    │    └── Sanitização de cabeçalhos, timeout e CORS       │
    │                                                        │
    │  [ systemd: cfa-ia-api.service ]                       │
    │    └── API FastAPI (127.0.0.1:8000)                    │
    │           ├── Qdrant Vector DB (127.0.0.1:6333)       │
    │           └── Ollama LLM Motor (127.0.0.1:11434)       │
    └────────────────────────────────────────────────────────┘
```

---

## 🛠️ Passo a Passo de Execução

---

### ETAPA 1: Automação Local no Linux via `systemd`

#### 🎯 O que é e por que fazer?
Atualmente, a API FastAPI precisa ser iniciada manualmente via SSH (`nohup python src/app.py &`).  
Se o servidor reiniciar por falta de energia, manutenção de kernel ou reboot preventivo, a API ficaria fora do ar até alguém logar no SSH.  
Com o serviço `systemd`:
* A API **sobe sozinha no boot** do sistema após o Ollama e o Docker estarem prontos.
* Se a API falhar ou travar por qualquer motivo, o `systemd` **reinicia o processo em 5 segundos automaticamente**.

#### 📋 Como configurar no `bulma`:
1. Conecte via SSH: `ssh root@bulma`.
2. Crie o arquivo do serviço:
   ```bash
   sudo nano /etc/systemd/system/cfa-ia-api.service
   ```
3. Cole a configuração abaixo:
   ```ini
   [Unit]
   Description=API FastAPI do Chatbot IA Local - CFA
   After=network.target ollama.service docker.service
   Wants=ollama.service docker.service

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/root/cfa-ia
   ExecStart=/root/cfa-ia/venv/bin/python /root/cfa-ia/src/app.py
   Restart=always
   RestartSec=5s
   Environment="PYTHONUNBUFFERED=1"
   Environment="OLLAMA_HOST=http://127.0.0.1:11434"
   Environment="QDRANT_HOST=http://127.0.0.1:6333"

   # Proteção e limites do processo
   LimitNOFILE=65536
   StandardOutput=append:/root/cfa-ia/api.log
   StandardError=append:/root/cfa-ia/api.log

   [Install]
   WantedBy=multi-user.target
   ```
4. Ative e inicie o serviço:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now cfa-ia-api.service
   ```
5. Valide se está rodando:
   ```bash
   sudo systemctl status cfa-ia-api.service
   curl http://127.0.0.1:8000/health
   ```

---

### ETAPA 2: Endereçamento IP e Segmentação de Rede (VLAN)

#### 🎯 O que é e por que fazer?
O servidor `bulma` não deve ficar na rede DHCP comum de estações de trabalho de usuários.  
* Se o IP mudar via DHCP dinâmico, o firewall e os proxies perdem a rota.
* A máquina deve ficar em uma **VLAN de Servidores / DMZ Interna**.
* O servidor **não precisa** ter rota de saída livre para a internet navegar, nem acesso às pastas de rede dos colaboradores. Precisa apenas receber o tráfego encaminhado pelo Sophos Firewall e poder baixar atualizações de sistema quando autorizado.

#### 📋 O que solicitar ao Técnico de Redes:
1. **Reserva de IP estático** para a interface de rede do `bulma` (via DHCP reservation por MAC Address ou IP fixo direto no `nmcli`).
2. Alocação na **VLAN de Serviços/Servidores**.
3. Garantir rota de comunicação entre o Sophos Firewall e esse IP na porta configurada.

---

### ETAPA 3: Firewall Local do Oracle Linux (`firewalld`)

#### 🎯 O que é e por que fazer?
Mesmo que a rede interna seja protegida, o firewall de host (`firewalld`) garante que nenhuma outra máquina da rede interna consiga acessar serviços sensíveis do servidor.

* As portas **`11434` (Ollama)** e **`6333` (Qdrant)** foram programadas para ouvir apenas em `127.0.0.1`.
* O `firewalld` deve abrir **apenas** a porta da API (TCP `8000` ou `443` se houver Nginx local), preferencialmente aceitando conexões **apenas vindas do IP interno do Sophos Firewall**.

#### 📋 Como configurar no `bulma`:

##### Opção Recomendada (Restrita ao IP do Firewall/Proxy):
Substitua `192.168.X.FIREWALL` pelo IP interno do Sophos Firewall:
```bash
# Permitir acesso à porta 8000 apenas a partir do Sophos Firewall
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.X.FIREWALL" port protocol="tcp" port="8000" accept'

# Manter SSH liberado apenas para a sub-rede de administração/TI
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.X.TI/24" service name="ssh" accept'

# Recarregar as regras
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

##### Opção Simplificada (Para testes iniciais na rede interna):
```bash
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload
```

---

### ETAPA 4: Publicação no Sophos Firewall (WAF, IPS e Rate Limit)

#### 🎯 O que é e por que fazer?
Esta é a barreira principal de proteção da IA do CFA perante a internet pública. O Sophos Firewall fará o papel de **WAF (Web Application Firewall)** e **Terminador SSL**.

#### 📋 Roteiro de Configuração no Sophos Firewall:

1. **Subdomínio Institucional:**
   * Criar o apontamento DNS externo (ex: `ia.cfa.org.br` ou `chatbot.cfa.org.br`) para o IP público do CFA no firewall.

2. **Certificado Digital SSL/TLS:**
   * Vincular o certificado institucional do CFA (wildcard `*.cfa.org.br` ou certificado emitido para o subdomínio) à regra de publicação HTTPS (porta 443).

3. **Regra de WAF (Web Server Protection):**
   * **Host de Origem:** IP público do subdomínio (`ia.cfa.org.br`).
   * **Host de Destino (Real Web Server):** IP interno do servidor `bulma` na porta `8000` (ou `443` se houver Nginx).
   * **Protocolo:** HTTPS externo -> HTTP interno (SSL Offloading).

4. **Proteção contra DoS e Limite de Requisições (Rate Limiting) — CRÍTICO:**
   * Como a GTX 1060 processa 1 inferência pesada por vez, **é mandatório** aplicar limitação de taxa no Sophos:
     * *Limite recomendado:* **10 requisições por minuto por IP de origem**.
     * *Conexões simultâneas máximas por IP:* **3 conexões**.
   * **Por que?** Evita que scripts maliciosos ou usuários abusivos esgotem a GPU e causem lentidão para os demais cidadãos.

5. **Perfil de IPS (Intrusion Prevention System):**
   * Ativar o perfil de IPS para tráfego web, bloqueando payloads maliciosos, scanners de vulnerabilidade e tentativas de injeção.

---

### ETAPA 5: Restrição de Domínio (CORS no Backend)

#### 🎯 O que é e por que fazer?
Atualmente, no ambiente de desenvolvimento local, a API está configurada com `allow_origins=["*"]` (aceita chamadas de qualquer lugar).  
Em produção, precisamos garantir que **apenas o portal oficial do CFA (`https://cfa.org.br`)** consiga consumir a API via JavaScript. Isso impede que terceiros usem a infraestrutura de IA do CFA dentro de sites não autorizados.

#### 📋 Ajuste no arquivo `src/app.py`:
Quando o domínio oficial estiver definido com a equipe de redes, alteramos em `src/app.py`:
```python
# Origens autorizadas estritas
ORIGENS_AUTORIZADAS = [
    "https://cfa.org.br",
    "https://www.cfa.org.br",
    "https://ia.cfa.org.br"
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ORIGENS_AUTORIZADAS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization"],
)
```

---

## 📊 Checklist Resumo para a Reunião com o Técnico de Redes

Leve esta lista impressa ou na tela para checar os itens com o colega:

| Item | Ação | Responsável | Status |
| :--- | :--- | :---: | :---: |
| **1. systemd** | Criar e habilitar `cfa-ia-api.service` no `bulma` | Infra / Erick | ⏳ A Fazer |
| **2. IP/VLAN** | Fixar IP estático no `bulma` em VLAN de Servidores | Redes | ⏳ A Fazer |
| **3. firewalld** | Liberar apenas TCP 8000 vindo do IP do Sophos | Infra / Redes | ⏳ A Fazer |
| **4. DNS** | Criar registro `ia.cfa.org.br` | Redes | ⏳ A Fazer |
| **5. Sophos WAF** | Publicar HTTPS (443) encaminhando para o `bulma:8000` com SSL | Redes | ⏳ A Fazer |
| **6. Rate Limit** | Travar em 10 req/min por IP no Sophos para proteger a GPU | Redes | ⏳ A Fazer |
| **7. CORS** | Travar origens em `src/app.py` apenas para domínios do CFA | Infra / Erick | ⏳ A Fazer |
| **8. Widget** | Inserir script do chat no portal do CFA | Web / TI | ⏳ A Fazer |
