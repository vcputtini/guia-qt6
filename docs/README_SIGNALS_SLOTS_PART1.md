# Qt Core: Entendendo o Conceito de SIGNALS & SLOTS – Parte 1

## 1) Event Loop: Por que as interfaces congelam e como o Qt resolve o caos assíncrono

### 1. O Problema Real

O fluxo de execução tradicional em C/C++ é linear e síncrono. Em um sistema operacional como o Linux, se você chama uma função demorada, como consultar um banco de dados gigante, copiar arquivos via rsync ou aguardar um socket TCP, de dentro do fluxo principal da interface gráfica, você bloqueia o 'Thread de Interface do Usuário' (UI Thread). O sistema operacional para de receber atualizações da tela, eventos de mouse e teclado, resultando no desagradável congelamento.

Aonde erramos: O erro mais comum é tentar “resolver” isso inserindo chamadas tais como `QcoreApplication::processEvents()` espalhas por todo o código dentro de loops de CPU, desse modo introduzindo condições de corrida (race conditions) bem estranhas, quebras de pilha de chamadas, reentrada indesejada de funções o que vai deixar o código extremamente complicado de depurar e manter.

### 2. Entendendo o Conceito

- Thread Principal (UI Thread): Um único garçom para atender todas as mesas.
- Loop de Eventos (QEventLoop): Nada mais é que o garçom circulando pelas mesas verificando se alguma chamando por ele. No sistema poderiam ser: cliques do mouse, toques no teclado, redimensionamento de janela, etc).

Desse modo podemos pensar em uma interface gráfica como um restaurante. O UI Thread é o único garçom do salão. Se o garçom anota o pedido de um cliente de bife bem passado e, em vez de passar a comanda para a cozinha, ele mesmo vai até a cozinha, acende o fogo, espera o bife grelhar por 15 minutos e volta, os outros clientes na mesa ficarão sem atendimento, ou seja, a interface congelou!

Qt6 resolve esse problema dividindo o restaurante em duas áreas bem definidas e integradas:

- Worker Threads (A Cozinha): São os cozinheiros em segundo plano. Eles preparam os pratos demorados (processamento de dados, arquivos e rede) sem nunca pisar no salão de atendimento.
- SIGNALS & SLOTS (A Campainha da Cozinha): Quando o bife fica pronto, a cozinha não vai até a mesa do cliente; ela simplesmente toca uma campainha (emit Signal). O garçom (UI Thread) escuta o aviso de imediato (Slot), pega o prato na bancada e entrega ao cliente de forma totalmente assíncrona!

```
                   SALÃO DO RESTAURANTE (UI Thread)
           +-------------------------------------------------+
           |              O Garçom (QEventLoop)              |
           |   [Clique de Mouse]  [Tecla]  [Redesenho Tela]  |
           +------------------------+------------------------+
                                    |
                    Passa a comanda | Toca a campainha
                     (Nova Thread)  | (Signals & Slots)
                                    v
                      COZINHA (Worker Threads)
           +-------------------------------------------------+
           |             Cozinheiros em Segundo Plano        |
           |   [Consulta BD]  [Download]  [Cálculo Pesado]   |
           +-------------------------------------------------+
```

```
+-------------------------------------------------------------+          
|                  O FLUXO DE EVENTOS (EVENT LOOP)            |          
+-------------------------------------------------------------+          
|                                                             |          
|   [ Teclado/Mouse ] -> Qt Event Queue                       |          
|                             |                               |          
|                             v                               |          
|                     +---------------+                       |          
|                     |  EVENT LOOP   | <---+ (Consome        |          
|                     | (exec() loop) |     |  constantemente)|          
|                     +---------------+     |                 |          
|                             |             |                 |          
|            +----------------+-------------+                 |          
|            |                                                |          
|            v                                                |          
|      +-----------+         (Sinal)         +-----------+    |          
|      | EMISSOR   | ======================> | RECEPTOR  |    |          
|      | (Botão)   |   (Disparado pelo MOC)  |  (Slot)   |    |          
|      +-----------+                         +-----------+    |          
|                                                             |          
+-------------------------------------------------------------+
```

### 3. Detalhamento Técnico

No núcleo do Qt está o **QEventLoop**, ativado quando seu main.cc executa `app.exec()`. Este loop monitora descritores de arquivos, sinais do sistema e eventos de periféricos no Linux (via epoll/glib). O sistema de SIGNALS e SLOTS é o mecanismo de acoplamento fraco (loose coupling) que substitui ponteiros de função brutos e callbacks inseguros. Um SIGNAL é apenas uma assinatura de método que informa: 'Algo aconteceu!'. Um SLOT é uma função normal que reage a isso: 'Deixe-me tratar esse evento!'. O acoplamento é fraco porque o emissor não faz ideia de quem está escutando, e o receptor não precisa saber quem disparou.

### 4. Arquiteturas de Código de Exemplo

#### A. Exemplo Simples: O Clássico Acoplamento de Elementos de UI

```
// main.cc - O jeito certo de acoplar elementos simples de UI        
        
#include <QApplication>          
#include <QPushButton>          
#include <QVBoxLayout>          
#include <QWidget>          
#include <QLabel>          
          
int main(int argc, char *argv[]) {          
    QApplication app(argc, argv);          
          
    QWidget window;          
    window.setWindowTitle("Controle de Backups");          
          
    QVBoxLayout *layout = new QVBoxLayout(&window);          
    QLabel *statusLabel = new QLabel("Status: Aguardando Comando", &window);          
    QPushButton *btnIniciar = new QPushButton("Iniciar Backup", &window);          
          
    layout->addWidget(statusLabel);          
    layout->addWidget(btnIniciar);          
          
    // Conexão: Quando o botão for clicado (Signal), altera o texto da label   (Slot)          
    // Usando a nova sintaxe de ponteiro de função do Qt5/Qt6          
    QObject::connect(btnIniciar, &QPushButton::clicked, [&]() {          
        statusLabel->setText("Status: Executando Backup em lote...");          
    });          
          
    window.show();          
    return app.exec(); // Inicia o Event Loop principal do Linux          
}
```

Neste exemplo, o clique no botão emite o sinal `clicked()`. O Qt captura esse sinal através de sua tabela interna do MOC e executa a função lambda que atua como Slot, atualizando o status da interface sem interromper o loop.

#### B. Exemplo Intermediário: Monitor de Processos Assíncrono com QProcess

```
// monitor.h - Monitorando logs do dmesg sem travar a tela          
#include <QWidget>          
#include <QProcess>          
#include <QTextEdit>          
#include <QPushButton>          
          
class SystemMonitor : public QWidget {          
    Q_OBJECT // MANDATÓRIO para habilitar Signals e Slots e metadados          
public:          
    SystemMonitor(QWidget *parent = nullptr);          
private slots:          
    void iniciarLeituraLogs();          
    void lerSaidaProcesso();          
    void processoTerminado(int exitCode);          
private:          
    QProcess *processoLinux;          
    QTextEdit *consoleLogs;          
    QPushButton *btnLer;          
};
```

A macro **Q_OBJECT** avisa ao compilador Qt (o MOC) que esta classe estende as capacidades nativas de reflexão do C++. Ela gera código para interceptar e despachar sinais de baixo nível do Linux.

#### C. Exemplo Avançado: O Padrão Profissional: Threads e Slots Seguros

```
// worker.h - Executando backups pesados em Thread separada via QueuedConnection          
#include <QThread>          
#include <QDebug>          
          
class BackupWorker : public QObject {          
    Q_OBJECT          
signals:          
    void backupProgresso(int porcentagem);          
    void erroOcorreu(QString msg);          
    void backupFinalizado();          
public slots:          
    void executarBackup() {          
        for(int i = 0; i <= 100; i += 10) {          
            QThread::msleep(200); // Simula leitura pesada de IO no Linux          
            emit backupProgresso(i); // Emite progresso com total segurança de thread          
        }          
        emit backupFinalizado();          
    }          
};
```

Ao emitir `backupProgresso(i)` de dentro de uma QThread paralela, o Qt detecta que o receptor do Slot está no Thread Principal e automaticamente engata uma Qt::QueuedConnection. O sinal é serializado como um QEvent e enfileirado com segurança na fila de mensagens do thread principal, evitando segfaults catastróficos.

### 5. Checklist de Homologação em Produção

- [ ] Certifiquei-me de que toda classe que usa signals/slots herda de QObject.

- [ ] Adicionei a macro Q_OBJECT no topo privado da definição da classe.

- [ ] Garanti que processos longos (IO/CPU) correm fora do Thread principal (UI Thread).

- [ ] Não usei chamadas abusivas de QCoreApplication::processEvents().

- [ ] Configurei meu sistema de build (CMake/qmake) para rodar o compilador MOC automaticamente.

### 6. FAQ de Suporte Técnico

**P: Por que eu recebo erros de compilador como 'undefined reference to vtable for MyClass'?** *R:* Isso ocorre porque você adicionou a macro Q_OBJECT à sua classe, mas não executou o MOC (Meta-Object Compiler). Se estiver usando CMake, garanta que set(CMAKE_AUTOMOC ON) esteja habilitado. Se estiver usando qmake, rode 'Clean' e depois 'Run qmake' novamente.

*Dica*: Por vezes é melhor fazer um ‘rm -rf build’ para garantir que nada fique em cache e atrapalhe uma nova compilação.

**P: Posso conectar um único sinal a múltiplos slots?** *R:* Sim! O Qt permite um mapeamento de 1 para N. Quando o sinal for emitido, os slots serão executados um após o outro na ordem em que foram conectados.

**P: Qual a diferença de performance entre Signals e Slots comparados a callbacks diretos?** *R:* Um Signal/Slot em conexão direta leva cerca de 10x mais tempo que um ponteiro de função puro (devido à resolução dinâmica dos metadados). Porém, em termos reais de sistemas, estamos falando de nanossegundos, o que acaba sendo imperceptível para interfaces humanas e extremamente seguro.

