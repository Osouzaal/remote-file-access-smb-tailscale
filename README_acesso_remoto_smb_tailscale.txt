====================================================================
 ACESSO REMOTO A SERVIDOR DE ARQUIVOS (SMB + TAILSCALE)
====================================================================

Documentação de uma solução de infraestrutura pessoal: acesso
remoto e seguro, de qualquer rede, a uma pasta de rede compartilhada
(SMB) rodando em um PC doméstico usado como servidor de arquivos
para laudos de raio-x.

Este repositório não contém código de aplicação — é a documentação
da arquitetura, das decisões técnicas e do passo a passo de
configuração, servindo como material de portfólio para demonstrar
conhecimento prático em redes, VPN mesh e compartilhamento de
arquivos.

--------------------------------------------------------------------
 PROBLEMA
--------------------------------------------------------------------

O trabalho envolve acessar e editar diariamente uma grande pasta de
laudos de exames de raio-x (PDFs, imagens, relatórios) a partir de
computadores diferentes, em locais e redes diferentes (ex: notebook
no local de trabalho, notebook em casa, outra rede Wi-Fi).

Sem uma solução de acesso remoto, isso exigiria:
  - Levar um pendrive/HD externo fisicamente entre os locais, ou
  - Depender de serviços de nuvem de terceiros (Google Drive,
    OneDrive, etc), com limite de espaço, custo mensal e menor
    controle sobre onde os dados médicos ficam armazenados, ou
  - Abrir portas do roteador para a internet (NAT/port forwarding),
    o que expõe o compartilhamento de arquivos publicamente e é
    um risco de segurança conhecido.

--------------------------------------------------------------------
 SOLUÇÃO: SMB (REDE LOCAL) + TAILSCALE (VPN MESH)
--------------------------------------------------------------------

A arquitetura usa duas camadas:

1) COMPARTILHAMENTO SMB (Server Message Block)
   Um PC em casa funciona como "servidor" de arquivos. A pasta com
   os laudos (ex: RAIOXSERVIDOR) é compartilhada na rede local via
   SMB, o mesmo protocolo nativo de "Compartilhar pasta" do Windows.
   Isso permite que outros computadores mapeiem essa pasta como uma
   unidade de rede comum (ex: Z:\), acessando e editando os arquivos
   como se estivessem no próprio disco local.

2) TAILSCALE (REDE PRIVADA VIRTUAL / VPN MESH baseada em WireGuard)
   Por padrão, o SMB só funciona dentro da mesma rede local (Wi-Fi/
   cabo). Para acessar de fora de casa sem abrir portas no roteador
   nem expor o compartilhamento à internet, os dispositivos (PC
   servidor + notebooks) são conectados a uma mesma "tailnet" —
   uma rede privada virtual criada pelo Tailscale.

   O Tailscale atribui um IP privado fixo (na faixa 100.x.x.x) a
   cada dispositivo da tailnet e cria um túnel criptografado
   ponto a ponto (WireGuard) entre eles, driblando NAT/CGNAT sem
   precisar mexer em configuração de roteador. Assim, de qualquer
   rede (trabalho, 4G, Wi-Fi de terceiros), o notebook enxerga o
   PC servidor pelo IP da tailnet como se estivesse na mesma rede
   local, e o compartilhamento SMB funciona normalmente.

--------------------------------------------------------------------
 FLUXO DE ACESSO
--------------------------------------------------------------------

  [Notebook em outra rede]
        |
        | túnel criptografado (Tailscale / WireGuard)
        v
  [Tailnet privada]
        |
        v
  [PC servidor em casa] -- compartilhamento SMB --> [Pasta RAIOXSERVIDOR]
        |
        v
  Unidade de rede mapeada (ex: Z:\) no notebook remoto,
  com leitura e escrita normais nos arquivos.

--------------------------------------------------------------------
 PASSO A PASSO (RESUMO DA CONFIGURAÇÃO)
--------------------------------------------------------------------

1. Instalar o Tailscale em todos os dispositivos (PC servidor e
   cada notebook que precisa acessar remotamente) e fazer login
   com a mesma conta, formando a tailnet.

2. No PC servidor (Windows):
   - Clique com o botão direito na pasta a compartilhar
     (ex: RAIOXSERVIDOR) -> Propriedades -> Compartilhamento ->
     Compartilhamento Avançado -> marcar "Compartilhar esta pasta".
   - Definir permissões de acesso (usuário/senha ou grupo
     específico, evitando "Todos" com controle total).
   - Garantir que o Firewall do Windows libere compartilhamento de
     arquivos e impressoras pelo menos para a interface de rede do
     Tailscale.

3. Verificar o IP da tailnet do PC servidor (comando `tailscale ip`
   ou pelo painel admin do Tailscale).

4. Nos notebooks remotos: mapear a unidade de rede usando o IP da
   tailnet do servidor, no formato:
     \\<IP-DA-TAILNET>\NomeDaPastaCompartilhada
   Isso pode ser feito por "Mapear unidade de rede" no Explorador
   de Arquivos do Windows, informando usuário/senha configurados no
   passo 2.

5. (Opcional) Configurar o Tailscale para iniciar automaticamente
   com o Windows em todos os dispositivos, garantindo que a unidade
   de rede esteja sempre disponível ao ligar o computador.

--------------------------------------------------------------------
 BENEFÍCIOS DESSA ABORDAGEM
--------------------------------------------------------------------

- Acesso e edição dos arquivos em tempo real de qualquer lugar,
  sem depender de sincronização (como em serviços de nuvem, que
  levam tempo para subir/baixar arquivos grandes).
- Nenhuma porta aberta no roteador / nenhuma exposição direta do
  compartilhamento à internet pública — o tráfego passa sempre por
  túnel criptografado ponto a ponto.
- Sem custo de armazenamento em nuvem de terceiros: os dados ficam
  fisicamente no próprio HD/servidor doméstico.
- Funciona de forma transparente: uma vez mapeada, a pasta remota
  se comporta exatamente como uma pasta de rede local para
  qualquer programa (Explorador de Arquivos, scripts de automação,
  editores de PDF, etc), sem precisar de nenhum cliente especial.

--------------------------------------------------------------------
 OBSERVAÇÕES E BOAS PRÁTICAS DE SEGURANÇA
--------------------------------------------------------------------

- Os IPs, nomes de pasta e caminhos usados em produção não são
  divulgados publicamente neste repositório — os exemplos aqui são
  genéricos e servem apenas para ilustrar a arquitetura.
- Como o compartilhamento envolve dados de exames médicos (dados
  sensíveis, sob LGPD), o acesso é restrito a usuário/senha
  específicos, e a rede Tailscale usada é privada (ACLs da tailnet
  configuradas para permitir conexão apenas entre os dispositivos
  autorizados).
- Recomenda-se manter o Tailscale, o Windows e o antivírus do PC
  servidor sempre atualizados, já que esse computador fica exposto
  como ponto de acesso remoto permanente.
- Um backup separado (independente deste compartilhamento) é
  mantido para os mesmos arquivos, seguindo o princípio de nunca
  depender de uma única cópia dos dados.

--------------------------------------------------------------------
 TECNOLOGIAS
--------------------------------------------------------------------

- SMB (Server Message Block) - protocolo de compartilhamento de
  arquivos nativo do Windows
- Tailscale - VPN mesh baseada em WireGuard, para conectividade
  ponto a ponto entre dispositivos sem exposição pública
- Windows (compartilhamento de pasta, mapeamento de unidade de
  rede, firewall)
