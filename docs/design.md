# Documento de design — Cordel (sincronização GNOME ↔ Android)

> Nome: **Cordel**
> Status: configuração do ambiente (nenhum código de produto escrito ainda)
> Última revisão: 28/10/2026

## 1. Visão

Um app de sincronização de arquivos entre um PC com GNOME e um celular Android, pela rede local, com barreira de uso mínima. Nativo nas duas pontas, publicável no **Flathub (Flatpak)** e na **Google Play Store**.

Nome e subtítulo (provisório): **Cordel — Sincronize seu celular com o GNOME**

O usuário nunca vê IDs, portas, servidores ou conceitos de protocolo. Pareia com um QR code, escolhe pastas e pronto.

## 2. Por que existe (lacuna de mercado)

| App | Sincroniza pastas | Nativo GNOME | Pareamento simples | Sem servidor para configurar |
|---|---|---|---|---|
| Syncthing | ✓ | ✗ (interface web) | ✗ | ✓ |
| FolderSync (Android) | ✓ | ✗ | ✗ (exige SMB/SFTP no PC) | ✗ |
| Packet / LocalSend / GSConnect / Warpinator | ✗ (só transferência) | parcial | ✓ | ✓ |
| **Cordel** | ✓ | ✓ | ✓ | ✓ |

Barreiras do Syncthing que motivam o projeto: configuração confusa (IDs de dispositivo e de pasta), interface web, conflitos como arquivos `.sync-conflict` soltos, versionamento desligado por padrão, app Android oficial descontinuado (dez/2024) e dependência de permissão "acesso a todos os arquivos", de aprovação incerta na Play Store.

Decisão: **implementação própria**, não baseada no Syncthing. Embutir o Syncthing exigiria esforço comparável e não resolveria a questão do armazenamento no Android.

## 3. Público e casos de uso

Público: usuários de Linux/GNOME que preferem não depender de nuvem, querem economizar com armazenamento pago ou valorizam privacidade. Nicho real e fiel; projeto open source, não negócio.

Casos de uso:
- Fotos do celular no PC sem nuvem, em qualidade original
- Liberar espaço no celular depois que as fotos chegaram ao PC
- Notas em Markdown (Obsidian, Logseq) iguais nos dois lados
- Saves de emuladores entre PC e portátil Android
- Música, podcasts, audiolivros e material de estudo do PC para o celular
- Documentos escaneados, comprovantes e gravações do celular para o PC

## 4. Os três modos de sincronização

Nomes claros em vez de um "espelho" genérico:

1. **Importar do celular** — celular → PC. Exclusões **nunca** se propagam: apagar do celular para liberar espaço não apaga do PC. *(MVP)*
2. **Enviar para o celular** — PC → celular. Espelho do PC.
3. **Manter igual** — bidirecional, para notas e saves.

## 5. Escopo e não-objetivos

Dentro:
- Pares PC ↔ celular (um celular pode ter vários PCs; cada par é independente)
- Somente rede local na primeira versão
- Endereço manual como recurso avançado (permite usar com VPN do próprio usuário, ex.: Tailscale)

Fora (conscientemente):
- Celular ↔ celular, PC ↔ PC, malha de vários dispositivos
- Sincronização pela internet com infraestrutura própria (ver seção 11)
- Armazenamento em nuvem
- Sincronização delta (transferir só o trecho alterado de um arquivo)
- iOS

## 6. Arquitetura

**O celular inicia, o PC decide.**

- **Celular (Kotlin + Jetpack Compose)**: cliente "burro" de propósito. Acorda (WorkManager), encontra o PC na rede, conecta e obedece comandos: listar, ler, gravar, apagar, renomear, criar pasta. Termina e dorme. Nenhum serviço permanente.
- **PC (Vala + GTK4 + libadwaita)**: serviço em segundo plano que espera conexões e concentra **toda** a lógica de reconciliação e todo o estado.

Vantagens: a lógica difícil existe uma vez só, testável sem celular; o app Android fica pequeno; não há núcleo compartilhado entre linguagens.

**Transporte separado do protocolo**: os comandos não sabem se estão sobre WebSocket na rede local ou sobre outro transporte no futuro. Rede remota deve ser "um novo transporte", nunca uma reescrita.

## 7. Algoritmo: reconciliação em três vias (estilo Unison)

Para cada par, o PC guarda o **estado da última sincronização bem-sucedida**, separadamente para cada lado: caminho relativo, tamanho, data observada e hash.

Cada lado é comparado com o *próprio* estado anterior, nunca com o outro lado. Isso torna irrelevante que o SAF não permita preservar datas de modificação no Android.

| PC | Celular | Ação (modo "Manter igual") |
|---|---|---|
| mudou | igual | copia para o celular |
| igual | mudou | copia para o PC |
| mudou | mudou | mesmo hash → só atualiza o estado; senão, **conflito** |
| apagou | igual | apaga no celular (via lixeira) |
| apagou | mudou | **mantém** a versão modificada |
| novo | novo | mesmo hash → nada; senão, conflito |

Regras complementares:
- Verificação rápida por tamanho + data; hash (SHA-256) só quando há dúvida
- Renomeação: arquivo "sumido" + arquivo "novo" com mesmo hash = mover, sem retransferir
- Conflito: mantém os dois; o perdedor recebe nome legível, ex.: `foto (conflito – celular 28-10).jpg`, e aparece numa tela de resolução
- Nos modos unidirecionais, a tabela se reduz ao lado de origem

## 8. Proteções contra perda de dados (não negociáveis)

- **Escrita atômica**: grava em temporário e renomeia ao final
- **Diário de operações**: estado atualizado arquivo a arquivo após sucesso; sincronização interrompida retoma com segurança
- **Lixeira com versões**: PC → pasta de versões; celular → pasta oculta dentro da pasta sincronizada; limpeza após 30 dias
- **Trava de exclusão em massa**: acima de um limite (ex.: 20% dos arquivos), para e pergunta. Protege contra permissão SAF revogada ou cartão SD removido, que fazem a pasta parecer vazia

## 9. Android

- **Armazenamento via Storage Access Framework (SAF)** desde o início, com permissões persistentes por pasta. Sem `MANAGE_EXTERNAL_STORAGE` (sincronização não está entre os usos listados como elegíveis pelo Google)
- Listagem via `DocumentsContract` + `ContentResolver` (não `DocumentFile`, lento demais)
- Pegadinhas do SAF:
  - Desde o Android 11, não é possível escolher a raiz do armazenamento nem a raiz de Download (subpastas sim)
  - Criar arquivos com MIME `application/octet-stream` para evitar extensões acrescentadas
  - Armazenamento compartilhado não diferencia maiúsculas/minúsculas (Linux diferencia) → avisar conflito
  - Cartões SD restringem caracteres `: * ? " < > |`
  - Não é possível definir data de modificação
- **Segundo plano via WorkManager**: tarefa periódica (mín. 15 min) com restrições ("só Wi‑Fi", "só carregando"); gatilho por novas mídias no MediaStore; transferências longas promovidas a serviço em primeiro plano do tipo "sincronização de dados" (atenção aos limites de tempo em versões recentes); botão "sincronizar agora"
  - Fabricantes com gerenciamento agressivo de bateria (Xiaomi/HyperOS, Samsung etc.) adiam ou cancelam tarefas do WorkManager. Prever tela de orientação para liberar execução em segundo plano, sem quebrar a barreira de uso mínima. Aparelho de referência nos testes: Android 16 com HyperOS (pior caso)
- Padrão: sincronizar só no Wi‑Fi
- Acompanhar a proteção de acesso à rede local que o Android vem introduzindo
- Play Store: formulário de segurança de dados simples (nenhum dado coletado, sem servidores)

## 10. GNOME / Flatpak

- Serviço (`GApplication` em modo serviço) separado da janela; iniciado com a sessão via portal de Background
- Notificações via `GNotification`
- Monitoramento com `GFileMonitor` + varredura periódica como garantia
- Pastas: portal de arquivos (correto para Flatpak, mas o monitoramento pode falhar no sistema de arquivos intermediário) **vs.** acesso direto a pastas padrão (mais simples, permissão mais ampla). **Testar cedo**
- Permissões: rede; acesso ao Avahi
- Estrutura: Meson, Blueprint para UI, `.desktop`, metainfo, GSchema, manifesto Flatpak
- Lógica de reconciliação em módulo puro (sem GTK), coberto por testes GLib

## 11. Rede e segurança

- **Descoberta**: PC anuncia via mDNS/DNS-SD (Avahi); Android encontra via `NsdManager`. Fallbacks: último IP conhecido e IP manual (redes de visitantes e alguns roteadores bloqueiam multicast)
- **Pareamento**: PC exibe QR code com endereço, porta, impressão digital do certificado e token de uso único
- **Transporte**: TLS com certificados autoassinados fixados no pareamento (autenticação mútua) + WebSocket por cima (libsoup no PC, OkHttp no Android). O celular conecta; o PC envia os comandos
- **Identidade por chave, nunca por IP**
- **Transferência** em blocos com retomada por posição
- **Protocolo tolerante a latência**: listar uma pasta é um pedido só, não um por arquivo

### Futuro: fora da rede local

Complicado mais por infraestrutura (servidores 24h, custo de banda dos relays, LGPD) do que por código. Escada de opções:

| Nível | Como | Custo contínuo |
|---|---|---|
| 0 | VPN do usuário (Tailscale) + endereço manual | zero |
| 1 | IPv6 direto + UPnP | zero, funciona às vezes |
| 2–3 | Iroh (QUIC, hole punching ~90%, relay de fallback) com relays próprios | baixo a médio |
| 4 | Armazenar na nuvem | outro produto |

Iroh 1.0 (jun/2026) tem bindings oficiais em Kotlin (compilar com NDK para Android); para Vala seria preciso uma camada Rust com API C + `.vapi`. Mesmo assim, os dois aparelhos precisam estar ligados ao mesmo tempo.

## 12. Roteiro

1. **Fase 1** — Protocolo + pareamento + modo "Importar do celular"
2. **Fase 2** — "Enviar para o celular" e "Manter igual" com estado em três vias, lixeira e trava de exclusão
3. **Fase 3** — Automação em segundo plano nas duas pontas
4. **Fase 4** — Renomeações, tela de conflitos, polimento, publicação (Flathub + Play Store)
5. **Depois** — Avaliar sincronização fora da rede local

## 13. Questões em aberto

- Formato das mensagens de controle (JSON ou CBOR)
- Versionamento e negociação de versão do protocolo (PC e celular atualizam em momentos diferentes)
- Onde e como guardar o estado no PC (SQLite? arquivo por par?)
- Portal de arquivos vs. acesso direto a pastas no Flatpak
- Interface: telas e fluxo de pareamento nas duas plataformas
- Formato dos vetores de teste em `protocol/`, consumidos pelos testes GLib e Kotlin (definir junto com o formato das mensagens)

## 14. Ambiente de desenvolvimento e convenções

- **Licença**: GPL-3.0-or-later (as duas pontas)
- **IDs**: `io.github.rochathili.Cordel` (Flatpak), `io.github.rochathili.cordel` (Android `applicationId`); repositório `github.com/rochathili/cordel`
- **Repositório**: monorepo
  - `desktop/` — Meson + Vala (app, serviço e biblioteca `core/` sem GTK, com testes GLib)
  - `android/` — projeto Gradle
  - `protocol/` — especificação das mensagens + vetores de teste compartilhados
  - `docs/` — documentação, incluindo este documento
- **Convenções**: código, nomes, comentários e mensagens de commit em inglês; commits com prefixo da área (`desktop:`, `android:`, `protocol:`, `docs:`)
- **PC**: desenvolvimento dentro do Flatpak desde o início, runtime `org.gnome.Platform//51`, GNOME Builder (Flathub); `blueprint-compiler` como módulo do manifesto; esqueleto inicial gerado pelo modelo do Builder
- **Android**: Android Studio oficial; `minSdk` 26 (Android 8.0), `targetSdk` 36 (exigência do Google Play desde 31/08/2026); esqueleto inicial gerado pelo modelo "Empty Activity" (Compose)
- **Testes**: rede (descoberta, pareamento, transferência) sempre com aparelho físico, já que o emulador fica atrás de NAT e não enxerga o mDNS do PC; emulador só para UI e SAF
