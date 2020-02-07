# Redes

# Multijogador de Alto Nível

# API de Alto nível vs baixo nível 

A seguir serão explicadas a diferença de rede de alto-nível e baixo-nível no Godot como também alguns fundamentos.

se você quer pular de cabeça e adicionar conexão a rede com seus nós, pule para [inicializando funcionamento de rede](#inicializando-funcionamento-em-redes) abaixo, Mas não deixe de ler o resto depois!

Godot sempre suportou padrões de funcionamento de rede de baixo nível(_low-level_) via _UDP_, _TCP_ e algumas de alto nível como protocolos _SSL_ e _HTTP_. estes protocolos são flexíveis e podem ser usados para quase qualquer coisa. no entanto forma usando eles para sincronizar estados de jogos manualmente pode ser bem trabalhoso.

Às vezes, esse trabalho não pode ser evitado ou vale mais a pena, por exemplo quando trabalhamos com um _servidor_ com implementações customizadas no _backend_. Mas na maioria dos vale a pena considerar a API de funcionamento de rede de nível-alto da Godot, que sacrifica alguns parte refinadas de controle de rede de _baixo-nível_ para maior facilidade de uso.

Isso ocorre devido às limitações dos protocolos de baixo-nível:

TCP garante que os pacotes sejam sempre entregue confiável e em ordem. mas a latência é geralmente maior devido a correção de erros. também é um protocolo um pouco complexo por que entende o que uma "conexão" é, e otimiza para objetivos que não se adéqua a aplicações como jogos multijogador.
Pacotes são _Buffered_ para ser enviados em grandes lotes, trocando menos sobrecarga por pacote por grande latência. Isso pode ser útil para coisas como HTTP, mas geralmente não para games. Parte disso pode ser configurado e desativado (exp: por desativar "Algorítimo de Nagle" pela conexão TCP).

UDP é um protocolo qual apenas envia pacotes(e não tem conceito de "conexão"),nenhuma correção de erro fazer isso ser rápido(baixa latência), mas pacotes podem ser perdidas ao longo do caminho ou recebidos na ordem incorreta. adicionando para que, O MTU (Tamanho Máximo do Pacote) para UDP é geralmente baixo (apenas algumas centenas de bytes), então transmitindo maiores pacotes significa dividi-los, reorganizá-los e tentar novamente se uma parte falha.

Em geral, TCP podem ser considerado confiável, ordenado e lento; UDP como não confiável, desordenado e rápido. Por causa da grande diferença em performance faz sentido reconstruir as partes do TCP desejadas para jogos (confiabilidade opcional e ordem nos pacotes) enquanto evitando as partes indesejadas (congestão/recursos de controle de tráfico, algoritmo de Nagle, etc). Devido a isso, a maioria dos _game engines_ vem com essas implementações, e Godot não é uma exceção.

	Em resumo você pode usar a API de funcionamento de rede de baixo-nível para o máximo controle e implementação tudo em cima de protocolos de rede simples ou usar a API de alto-nível baseada em [SceneTree]() que faz o trabalho mais pesado nos bastidores de uma maneira geralmente otimizada.
---
 Nota

	|Maioria das plataformas suportadas pelo Godot oferecem todas ou maioria dos recursos de rede de baixo-nível e alto-nível. Como a rede é sempre amplamente dependente de _hardware_ e sistemas operacionais, contudo, alguns recursos pode mudar ou não sendo disponível em algumas plataformas almejadas. Mais notável,a plataforma HTML5 atualmente apenas oferece WebSocket suporte e não possui alguns dos recursos de alto-nível, bem como acesso bruto a protocolos de baixo-nível, como TCP e UDP.|

---

---
 Nota

	Mais sobre TCP/IP, UDP e redes: https://gafferongames.com/post/udp_vs_tcp/

	Gaffer On Games tem um monte de artigos úteis sobre redes em games ([aqui](https://gafferongames.com/tags/networking)), incluindo a compreensiva [introdução de redes e modelos em jogos.](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/).

	se você quer usar sua biblioteca de redes de baixo-nível de escolha do que a de redes, interna da Godot, veja aqui por exemplo: https://github.com/PerduGames/gdnet3 
---

---
 Atenção
	
	Adicionar funcionamento de redes em seus jogos vem com grande responsabilidades. Isso pode fazer sua aplicação vulnerável se feito errado e pode levar a trapaças e _exploits_. Pode até permitir um ataque para comprometer as maquinas onde suas aplicações rodam e usar seus servidores para mandar spam, e atacar ou roubar dados de outros usuários se eles jogarem seu jogo.

	Isso é sempre o caso quando a rede está envolvida e não tem nada para fazer com o Godot.
	Você pode é claro experimentar, mas quando você lança uma aplicação de rede, sempre cuide de quaisquer possíveis preocupações de segurança.

---
# Abstração de nível médio
Alguns jogos aceitam conexões no mesmo tempo, outros durante o faze de _lobby_. Godot pode ser requisitado a não aceitar conexões a qualquer momento. (veja _set_refuse_new_network_connections(bool)_ e métodos no [SceneTree](http://docs.godotengine.org/en/3.0/classes/class_scenetree.html#class-scenetree)). Para gerenciar quem conecta, Godot fornece sinais na SceneTree:

Servidor e Cliente:

. _network_peer_connected(int id)_
. _network_peer_disconnected(int id)_

Os sinais a cima são chamados a cada _par_ conectado ao servidor (incluindo no servidor) quando um novo _par_ conecta ou desconecta. Clientes iram conectar com uma única ID maior do que 1, sendo o servidor sempre será o _peer_ com ID 1. qualquer um abaixo de 1 devera ser manipulado como inválido. você pode recuperar a ID para o sistema local via [SceneTree.get_network_unique_id()](http://docs.godotengine.org/en/3.0/classes/class_scenetree.html#class-scenetree-get-network-unique-id).
Esses IDs pode serem uteis principalmente para gerenciar e podem geralmente ser armazenados como eles identificam _pares_ conectados e ,portanto, jogadores. Você também pode usar IDs para enviar mensagens para determinados _pares_.

Clientes:


. connected_to_server
. connection_failed
. server_disconected

Novamente, todos essas funções são principalmente uteis para gerenciamento de sala de espera ou para adicionar/remover jogadores em movimentos. Para esses tarefas o servidor claramente tem que trabalhar como um servidor e você tem tarefas manuais como o envio de informações sobre o jogador e outros já conectados. (exp: seus nomes, stados, etc).

Salas de espera podem ser implementadas na maneira que você quiser, mas a maneira mais comum é usar um nó com o mesmo nome em todos os _pares_. Geralmente, um nó/_singleton_ carregado automaticamente é um bom ajuste para isto, para sempre ter acesso a, exp: "/root/lobby".

# RPC

Para comunicar entre pares, o caminho mais fácil é usar RPCs (chamadas procedurais remotas). isto é implementado com a colocação de funções no [Nó](http://docs.godotengine.org/en/3.0/classes/class_node.html#class-node):

. rpc(“nome_da_função”, <argumento_opcional>)
. rpc_id(<peer_id>,nome_da_função”, <argumento_opcional>)
. rpc_unreliable(nome_da_função”, <argumento_opcional>)
. rpc_unreliable_id(<peer_id>, nome_da_função”, <argumento_opcional>)

Também é possível sincronizar variáveis de membro:


. rset(“variavel”, valor)
. rset_id(<par_id>, “variavel”, valor)
. rset_unreliable(“variavel”, valor)
. rset_unreliable_id(<par_id>, “variavel”, valor)

As funções podem ser chamadas em duas maneiras:
. Confiável: a função chama chegara não importa o que seja, mas pode levar mais tempo porque será retransmitido em caso de falha.

. Não confiável: se a chamada de função não chegar, não ira ser retransmitida, mas se chegar, o fará rapidamente.

Na maioria dos casos, confiável é desejado. Não confiável é mais útil quando sincroniza posições dos objeto (sincronização pode acontecer constantemente, e se um pacote é perdido, não é ruim porque um outro sera eventualmente entregue, e pode estar desatualizado porque o objeto movido mais longe em quanto isso, mesmo se fosse reenviado de forma confiável).

Há também a função get_rpc_sender_id na cena SceneTree que pode ser usado para checar qual _par_ (ou ID do par) envia uma chamada RPC.

# Back to lobby
Vamos voltar para o salão de entrada. Imagine que cada jogador que conectar ao servidor ira falar para todos.

``` GDScript
# Implementação tipica de sala de espera, imagine isto em /root/lobby

extends Node

# Conecta todas funções

func _ready():
    get_tree().connect("network_peer_connected", self, "_jogador_conectado")
    get_tree().connect("network_peer_disconnected", self, "_jogador_disconectado")
    get_tree().connect("connected_to_server", self, "_ok_conectado")
    get_tree().connect("connection_failed", self, "_conexao_falhou")
    get_tree().connect("server_disconnected", self, "_servidor_desconectado")

# Informação do jogador, associado a uma ID
var info_do_jogador = {}
# Informação que mandamos para outros jogadores
var minha_info = { name = "Joãozinho HU3", favorite_color = Color8(0, 0, 255) }

func _jogador_conectado(id):
    pass # Não sera usado, pouco util aqui nesse exemplo.

func _jogador_disconectado(id):
    info_do_jogador.erase(id) # apaga informação do jogador

func _ok_conectado():
    # Apenas chamado no Clientes, não no server. Envia minha ID e minha info para todos outros players
    rpc("registrar_jogador", get_tree().get_network_unique_id(), minha_info)

func _servidor_desconectado():
    pass # Server kickou a gente, mostra uma mensagem de abortagem

func _conexao_falhou():
    pass # Não foi possível conectar ao servidor, abortagem

remote func registrar_jogador(id, info):
    # Armazena a informação
    info_do_jogador[id] = info
    # Se eu sou o servidor, informa o cara novo sobre os outros jogadores
    if get_tree().is_network_server():
        # Envia minha informação para o player
        rpc_id(id, "registrar_jogador", 1, minha_info)
        # Envia informação dos players existentes
        for peer_id in info_do_jogador:
            rpc_id(id, "registrar_jogador", peer_id, info_do_jogador[peer_id])

    # Chame a função para atualizar a informação da interface de usuário da sala de espera aqui

```
Você ja pode ter natado algo diferente, qual é o uso da palavra-chave _remote_ na função _registrar_jogador_:

```GDScript
 remote func registrar_jogador(id, info): 
```

Esta palavra-chave tem dois usos principais, O primeiro é para deixar a Godot saber que esta função pode ser chamada pelo RPC. se nenhuma palavras-chaves for adicionado, Godot ira bloquear qualquer tentativas de chamar a função por segurança. Isto facilita muito o trabalho de segurança( então um cliente não pode chamar uma função para deletar um arquivo em um sistema de outro cliente).

A segundo uso é para especificar como a função poderá ser chamada via RPC. existem quatro palavras-chaves:

. remote
. sync
. master
. slave

A palavra-chave _ remote_ significa que o _rpc()_ poderá ir pela rede e executar remotamente.

O palavra-chave _sync_ significa que o _rpc()_ poderá ir pela rede e executar remotamente, mas ira também executar localmente (faça uma chamada de função normal).

Os outros serão explicados mais a baixo. Note que você pode também usar a função _get_rpc_sender_id()_ na _SceneTree_ para checar qual _par_ atualmente feito com RPC chama o _registrar_jogador()_.

Com isto, o gerenciador do salão de espera pode ser mais ou menos explicado. Uma vez que seu jogo começa, você adicioná funções extras de segurança para ter certeza que os clientes não façam nada engraçado(Apenas valide a informação deles tempo em tempo, ou depois do jogo começar). Por uma questão de simplicidade e porque cada jogo poderá compartilhar diferentes informações, isto não é mostrado aqui.

# Começando o jogo

Uma vez que jogadores suficiente entraram no salão de espera, o servidor poderá provavelmente começar o jogo. Isto é nada especial em si, mas iremos explicar alguns truques legais que podem ser feito nesse momento para fazer sua vida mais fácil.

## Cenas dos jogador

Em vários jogos, cada jogador ira provavelmente ter sua própria. Lembre que isto é um jogo multijogador, então cada _par_ seu precisa ser instanciado para isso **uma cena para cada jogador conectado**. Para um jogo de 4 jogadores, cada _par_ precisa instanciar 4 nós dos jogadores.

Então, como nomear tantos nós? na Godot precisamos ter um nome único. precisa também ser relativamente fácil para um jogador falar qual nós representam cada ID de jogador.

A solução é simplesmente nomear a raiz(_root_) dos nós da cena do jogador instanciado com seu ID de rede. Desta forma, Eles serão os mesmo em cada _par_ e RPC, funcionara muito melhor! aqui um exemplo:

```GDScript
remote func pre_configurar_o_jogo():
    var propioPeerID = get_tree().get_network_unique_id()

    # Carrega o mundo
    var mundo = load(which_level).instance()
    get_node("/root").add_child(mundo)

    # Carrega meu jogador
    var meu_jogador = preload("res://jogador.tscn").instance()
    meu_jogador.set_name(str(propioPeerID))
    meu_jogador.set_network_master(propioPeerID) # Sera explicado depois
    get_node("/root/mundo/jogadores").add_child(meu_jogador)

    # Carrega os outros jogadores
    for p in info_do_jogador:
        var jogador = preload("res://jogador.tscn").instance()
        jogador.set_name(str(p))
        get_node("/root/mundo/jogadores").add_child(jogador)

    # Diga ao servidor que esse _par_ é feito pré-configurando (Lembre, o servidor sempre é o ID=1).

    rpc_id(1, "feito_preconfig", propioPeerID)

```

---
 Nota

	Dependendo de quando você executar pre_configurar_o_jogo() você pode ser necessário mudar algumas chamadas para `` add_child() `` para serem adiadas via ``call_deferred()`` como a SceneTree é bloqueada enquanto a cena começa a ser criada(Exp: quando ``_reandy()`` está sendo chamado).
---
# Sincronizando o inicio do jogo

A configuração do jogador pode levar quantidades de tempos diferentes devido ao lag, diferentes _hardwares_, ou outras razões. Para garantir que o jogo começara realmente quando todos estiverem prontos, pausando o game até que todos jogadores estejam prontos pode ser útil:

```GDScript
remote func pre_configurar_o_jogo():
    get_tree().set_pause(true) # Pre-pausar
    # O resto é o mesmo como no código da seção anterior.(Olhe a cima)
```

Quando o servidor obtêm OK de todos os _pares_, pode dizer para eles começarem, como por exemplo:

```GDScript
var jogadores_prontos = []
remote func feita_pre_configuracao(who):
    # Aqui estão algumas verificações que você pode fazer, por exemplo
    assert(get_tree().is_network_server())
    assert(who in player_info) # Existe
    assert(not who in jogadores_prontos) # ainda não foi adicionado

    jogadores_prontos.append(who)

    if jogadores_prontos.size() == player_info.size():
        rpc("pos_configurar_jogo")

remote func pos_configurar_jogo():
    get_tree().set_pause(false)
    # O Jogo começa agora!!
```

## Sincronizando o jogo

Em vários jogos a meta do multijogador em rede é que o jogo rode sincronizado em todos _pares_ que estão jogando. Além de fornecer um RPC e implementação de definir membros de variáveis, Godot adiciona o conceito _network masters_ (mestres da rede).

## Mestre da rede

O mestre da rede de um nó é o _par_ que tem a autoridade supre sobre.

Quando não explicitamente configurada o mestre da rede é herdado do nó pai, que se não mudado, sempre sera o servidor (ID 1), Portanto o servidor sempre tera autoridade sobre todos os outros nós por padrão.

O mestre da rede pode ser configurado com a função [Node.set_network_master(id, recursive)](http://docs.godotengine.org/en/3.0/classes/class_node.html#class-node-set-network-master). (recursivo é _true_ por padrão e significa que o _network master_ é definido de forma recursiva em todos nós filhos do nó também).


Verificando se um nó instanciado especifico em um _par_ é o _network master_ para este nó para todos _par_ conectos é feita chamando [Node.is_network_master().](http://docs.godotengine.org/en/3.0/classes/class_node.html#class-node-is-network-master).Isso ira retornar _true_ quando executa no servidor e _false_ em todos os clientes.

se prestou atenção no exemplo anterior, possivelmente você notou que o _par_ local é configurado para ter autoridade de mestre da rede para seu próprio jogador instanciado no servidor: 

``` GDScript
[...]
# Carrega meu jogador
var meu_jogador = preload("res://jogador.tscn").instance()
meu_jogador.set_name(str(propioPeerID))
meu_jogador.set_network_master(propioPeerID) 
# O jogador pertence a este par, tem a autoridade

get_node("/root/mundo/jogadores").add_child(meu_jogador)

# Carrega outros jogadores
for p in player_info:
    var jogador = preload("res://jogador.tscn").instance()
    jogador.set_name(str(p))
    get_node("/root/mundo/jogadores").add_child(jogador)
[...]
```

cada vez que esse pedaço de código é executado em cada _par_, o _par_ torna ele mesmo no mestre no nó que controla, e todos outros nós permanecem como subordinados, com o servidor sendo seu seu mestre da rede. 

Para deixar mais claro, aqui um exemplo de como isto parece no [bomber demo](https://github.com/godotengine/godot-demo-projects/tree/master/networking/multiplayer_bomber)

![]("/_images/nmms.png")

## Metre e _subordinados_ palavras-chave

A real vantagem desse modelo é quando usado com o _mestre/subordinado_ no GDScript (ou seus equivalentes em C# e Visual Script). Similar a palavra chave _remote_, funções podem também ser marcadas com elas: 

Exemplo do código _bomb_:
```GdScript
for p in corpos_na_area:
	if p.has_method("explode"):
		p.rpc("explodiu", dono_da_bomba)
```
Exemplo do código do jogador:

```GDScript
slave func atordoar():
    atordoado = true

master func explodiu(por_quem):
    if atordoado:
        return # Já atordoado

    rpc("atordoar")
    atordoar() # Me atordoa, poderia também usar a palavra-chave de sincronização.
```

No exemplo acima, a bomba explode em algum lugar (provavelmente gerenciada por quem é o mestre).
A bomba sabe os corpos que estão na área, então checa que eles contem um função _explodiu_.

se a fizeram, a bomba vai chamar _explodiu_ nela, contudo, o método _explodiu_ no jogador tem uma palavra-chave _mestre_, Isto significa que apenas o jogador  que é mestre para está instancia, realmente recebera a função.

Essa instância, então, chamara a função _atordoar_ na mesma instância do mesmo jogador(mas em diferentes _pares_), e apenas aqueles que são configurados como subordinados, fazendo o jogador parecer stunado em todos os _pares_ (bem como o atual, mestre).

Note que você pode também enviar mensagem _atordoar()_ para um jogador especifico usando rpc_id(<id>, “explodiu”, dono_da_bomba). isso pode não fazer sentido para uma área de efeito, como a bomba em outros exemplos, como dano de alvo único.

```GDScript
rpc_(TARGET_PEER_ID, "stun") # Apenas atordoa o par alvo

```
