# Qt Core: Entendendo o Conceito de SIGNALS & SLOTS – Parte 3

## 3)  Custom Signals & Slots

### Dando voz aos seus objetos de sistema com a macro Q_OBJECT e herança pura de QObject

### 1. O Problema Real

Escrever classes C++ normais que precisam notificar outras sem criar dependências circulares estreitamente acopladas. Por exemplo, se a classe `DiskMonitor` precisar chamar uma função de `MailNotifier` de forma estática, a classe `DiskMonitor` agora passa a depender e compilar junto com `MailNotifier`. Se amanhã você quiser remover a notificação por e-mail, terá que reescrever e recompilar a classe principal.

*Onde erramos:* Criar heranças monstruosas ou passar ponteiros globais de instâncias ('singletons' abusivos) para resolver o problema de comunicação entre módulos independentes, o que gera o pesadelo do código espaguete.

### 2. Entendendo o Conceito

**A Linha de Montagem Modular** Imagine uma linha de montagem de veículos. O operário que coloca a porta do carro (Emetor de sinal de porta finalizada) não precisa conhecer quem vai pintar o carro (Slot de pintura) ou quem vai instalar o motor. Ele apenas coloca um adesivo verde na lataria ('Porta Instalada!'). O supervisor da linha de montagem (o Qt Connection Manager) vê o adesivo verde e chama o pintor de portas. Se você mudar a máquina de pintura para uma robótica, o instalador de portas nem nota a diferença — os módulos são 100% independentes.

```
          
+-------------------------------------------------------------+          
|               ARQUITETURA DE COMUNICAÇÃO DE DADOS           |          
+-------------------------------------------------------------+          
|                                                             |          
|   +-------------------+                                     |          
|   |   DiskMonitor     | <-- Monitora discos na máquina      |          
|   +-------------------+                                     |          
|             |                                               |          
|             | (Emite sinal diskLimitReached(percent, "/"))  |          
|             v                                               |          
|      [ QT DISPATCHER (MOC) ]                                |          
|             |                                               |          
|             +-----------------------+-----------------------+          
|                                     |                       |          
|                                     v                       v          
|                            +-----------------+     +-----------------+          
|                            |   LogCleaner    |     |   MailNotifier  |          
|                            | (Executa Slot)  |     | (Executa Slot)  |          
|                            +-----------------+     +-----------------+          
|                                                             |          
+-------------------------------------------------------------+
```

### 3. Detalhamento Técnico

Para implementar Signals ou Slots customizados em sua própria classe, ela precisa obrigatoriamente herdar de QObject (ou de qualquer classe derivada como QWidget ou QMainWindow) e possuir a macro `Q_OBJECT` no início de sua declaração corporativa privada. O compilador MOC varre esta classe e gera funções de suporte que registram o sinal nos metadados do Qt. Sinais são definidos sob a diretiva de acesso `signals:` e nunca devem possuir um corpo/implementação em seu arquivo .cc — o MOC gera a implementação para você. Slots são definidos sob a diretiva `public slots:` ou simplesmente declarados como métodos normais e implementados livremente.

### 4. Arquiteturas de Código de Exemplo (Bret Fisher & Goasguen Style)

#### A. Exemplo Simples: Definição Completa do DiskMonitor

```
// diskmonitor.h - Declarando sinais customizados          
#ifndef DISKMONITOR_H          
#define DISKMONITOR_H          
          
#include <QObject>          
#include <QString>          
          
class DiskMonitor : public QObject {          
    Q_OBJECT // Habilita metadados e infraestrutura de Signals/Slots          
public:          
    explicit DiskMonitor(QObject *parent = nullptr) : QObject(parent) {}          
          
    void verificarUso(int usoPercentual) {          
        if (usoPercentual >= 90) {          
            // Emissão do sinal com a palavra-chave 'emit'          
            emit limiteAtingido(usoPercentual, "/dev/sda1");          
        }          
    }          
          
signals:          
    // SINAIS NÃO POSSUEM CORPO! São apenas declarados.          
    void limiteAtingido(int percentual, const QString &particao);          
};          
          
#endif
```

Note a palavra chave `emit`. Ela é uma macro vazia em termos de C++, mas funciona como um marcador semântico essencial para leitura humana e para que os depuradores de código saibam que um sinal está sendo disparado.

#### B. Exemplo Intermediário: A Classe Receptora (Slot)

```
// logcleaner.h - Executando a resposta ao alerta          
#ifndef LOGCLEANER_H          
#define LOGCLEANER_H          
          
#include <QObject>          
#include <QDebug>          
          
class LogCleaner : public QObject {          
    Q_OBJECT          
public:          
    explicit LogCleaner(QObject *parent = nullptr) : QObject(parent) {}          
          
public slots:          
    // Slots são implementados normalmente no .cc          
    void realizarLimpeza(int percentual, const QString &particao) {          
        qDebug() << "ALERTA: Partição" << particao << "com" << percentual << "% de uso!";          
        qDebug() << "Ação Automática: Removendo arquivos temporários em /var/log...";          
    }          
};          
          
#endif
```

Os slots recebem exatamente os mesmos argumentos enviados pelo sinal correspondente, permitindo o tráfego limpo e tipado de informações estruturadas de sistema.

#### C. Exemplo Avançado: Amarrando o Sistema no main.cpp

```
// main.cpp - Orquestração robusta de sinais acoplados          
#include <QCoreApplication>          
#include <QTimer>          
#include "diskmonitor.h"          
#include "logcleaner.h"          
          
int main(int argc, char *argv[]) {          
    QCoreApplication app(argc, argv);          
          
    DiskMonitor monitor;          
    LogCleaner faxineiro;          
          
    // Acoplamento fraco impecável!          
    QObject::connect(&monitor, &DiskMonitor::limiteAtingido,          
                     &faxineiro, &LogCleaner::realizarLimpeza);          
          
    // Simula a passagem do tempo e verificação ativa do sistema          
    QTimer::singleShot(1500, [&]() {          
        monitor.verificarUso(95); // Vai disparar o sinal          
    });          
          
    QTimer::singleShot(3000, &app, &QCoreApplication::quit); // Finaliza o app de teste          
          
    return app.exec();          
}
```

A orquestração isola os componentes. Você pode plugar qualquer outro receptor ao sinal `limiteAtingido` sem modificar uma única linha sequer do arquivo `diskmonitor.h`.

### 5. Checklist de Homologação em Produção

- [ ] Incluí a macro Q_OBJECT no topo da minha classe.

- [ ] Garanti que herdei direta ou indiretamente da classe base QObject.

- [ ] Não criei qualquer corpo de função para o método do Signal.

- [ ] Usei os tipos nativos do Qt (QString, QList) ou registrei minhas structs personalizadas com qRegisterMetaType.

- [ ] Usei o ponteiro parent para manter a árvore de ciclo de vida de memória segura.

### 6. FAQ de Suporte Técnico

**P: Por que não posso instanciar o corpo de uma função signal?** *R:* Porque o compilador MOC precisa reescrever o sinal de forma a empacotar os parâmetros em estruturas internas de array de ponteiros genéricos (`void*`) para distribuí-los para os slots cadastrados. Se você criar um corpo para ele, o linker do C++ apontará conflito de funções duplicadas.

**P: Posso passar meus próprios tipos struct personalizados nos sinais e slots?** *R:* Sim! Mas para que o Qt consiga propagar tipos customizados (principalmente em conexões de threads diferentes), você precisa registrar o tipo com o Qt Type System usando `qRegisterMetaType<MinhaStruct>("MinhaStruct")` antes de fazer a conexão.

**P: O que é o parâmetro 'parent' (QObject parent) usado no construtor?* *R:* Ele gerencia a árvore hierárquica de memória do Qt. Quando o objeto pai é deletado da memória, todos os seus filhos são automaticamente e recursivamente deletados também, prevenindo vazamentos de memória (Memory Leaks) brutais comuns em C++.

