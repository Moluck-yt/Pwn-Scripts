# Exploit: Desserialização Insegura via PHAR Upload

Este script (`generate_phar.php`) cria um arquivo PHAR malicioso para explorar uma vulnerabilidade de Desserialização Insegura em aplicações PHP. O objetivo é realizar um upload de arquivo que, ao ser processado por uma funcionalidade vulnerável (como um download), escreve um _web shell_ em um diretório acessível, resultando em Execução Remota de Código (RCE).

## Análise da Vulnerabilidade

A exploração se baseia na combinação de duas condições em uma aplicação PHP:

1.  **Um "Sink" de Arquivo:** A existência de uma função de sistema de arquivos (como `file_get_contents`, `filesize`, `readfile`, `file_exists`, etc.) que opera em um caminho de arquivo controlado pelo usuário.
2.  **Um "Gadget Chain":** A presença de uma ou mais classes no código da aplicação que, quando desserializadas, podem ser abusadas para realizar ações inesperadas através de seus "métodos mágicos" (ex: `__destruct`, `__wakeup`).

### O Gatilho: Wrapper `phar://`

O núcleo da vulnerabilidade reside em como o PHP lida com arquivos PHAR (PHP Archive). Quando qualquer função de sistema de arquivos tenta acessar um arquivo usando o wrapper `phar://`, o PHP automaticamente lê os metadados do arquivo PHAR.

Esses metadados são armazenados em um formato serializado. Ao lê-los, o PHP **desserializa** os dados, o que significa que ele recria os objetos PHP que estavam armazenados ali. Se um atacante puder controlar o conteúdo desses metadados, ele pode instanciar objetos de classes arbitrárias que existem no escopo da aplicação.

No código-alvo, a funcionalidade de download é o nosso **Sink**:

```php
// ...
if (strpos($file,'phar://')===0) {
    $path = $file;
}
// ...
header('Content-Length: '.filesize($path)); // <-- GATILHO
readfile($path);                           // <-- GATILHO
```


A aplicação explicitamente confia e processa caminhos com `phar://`. Quando `filesize()` ou `readfile()` são chamados em um caminho como `phar://path/to/uploaded/file.jpg`, a desserialização é acionada.

### O Gadget Chain: A Classe `LogManager`

Um "gadget" é uma classe que possui métodos mágicos que podem ser abusados. Neste cenário, a classe `app\classes\LogManager` é o nosso gadget perfeito.

```php
class LogManager
{
    public $path = '/tmp/';
    public $file = 'log.txt';
    public $content;

    // ...

    public function __destruct()
    {
        file_put_contents($this->path . $this->file, $this->content);
    }
}
```

O método mágico chave aqui é o `__destruct()`. Ele é executado automaticamente quando um objeto dessa classe é destruído (por exemplo, no final da execução de um script). A função `file_put_contents` dentro dele escreve um arquivo no servidor.

Crucialmente, as propriedades `$path`, `$file` e `$content` são públicas. Isso significa que, ao criar o objeto `LogManager` para ser desserializado, um atacante pode controlar **todos os três argumentos** da função `file_put_contents`, nos dando uma capacidade de **escrita de arquivo arbitrária**.

## Estratégia de Exploração

O ataque ocorre em três etapas:

1.  **Geração do Payload:** Executamos o script `generate_phar.php`. Ele faz o seguinte:

    - Cria um novo arquivo PHAR (`payload.phar`).
    - Instancia um objeto da classe `app\classes\LogManager`.
    - Define as propriedades do objeto:
      - `$path`: O caminho no servidor para o diretório de uploads (ex: `/var/www/html/uploads/`).
      - `$file`: O nome do nosso web shell (ex: `revshell.php`).
      - `$content`: O código do nosso web shell (ex: `<?php system($_GET["cmd"]); ?>`).
    - Armazena este objeto malicioso e configurado nos metadados do arquivo PHAR.

2.  **Upload do Payload:** A aplicação possui uma funcionalidade de upload de imagens. Nós fazemos o upload do nosso `payload.phar`, mas o renomeamos para uma extensão de imagem permitida (ex: `payload.jpg`) para contornar validações de tipo de arquivo. O servidor armazena o arquivo, por exemplo, em `/var/www/html/uploads/payload.jpg`.

3.  **Ativação do Payload:** Fazemos uma requisição para a funcionalidade de download vulnerável, usando o wrapper `phar://` para apontar para o nosso arquivo recém-enviado:

    ```
    [https://vulnerable-app.com/download.php?f=phar://uploads/payload.jpg](https://vulnerable-app.com/download.php?f=phar://uploads/payload.jpg)
    ```

    O servidor tenta ler o arquivo. O `filesize()` aciona o wrapper `phar://`, que por sua vez desserializa o objeto `LogManager` dos metadados. Ao final da requisição, o método `__destruct` é chamado, e nosso web shell é escrito em `/var/www/html/uploads/revshell.php`.

4.  **Execução de Código:** Agora, basta acessar o shell para executar comandos no servidor:

    ```
    [https://vulnerable-app.com/uploads/revshell.php?cmd=whoami](https://vulnerable-app.com/uploads/revshell.php?cmd=whoami)
    ```

## Como Usar o Script Gerador

1.  Certifique-se de que seu ambiente PHP CLI tem a extensão `phar` habilitada e `phar.readonly` está desativado no `php.ini` (o script tenta fazer isso em tempo de execução com `ini_set`).
2.  Ajuste as variáveis no script, principalmente `$web_shell_path`, para o caminho absoluto correto do diretório de uploads do alvo.
3.  Execute o script: `php generate_phar.php`.
4.  O arquivo `payload.phar` será criado no mesmo diretório.

## Mitigação da Vulnerabilidade

- **Não confie em input do usuário:** Nunca passe caminhos de arquivo controlados pelo usuário diretamente para funções de sistema de arquivos.
- **Desabilitar `phar://`:** Se não for essencial para a aplicação, desabilite o wrapper `phar://` no `php.ini`.
- **Verificação de Assinatura:** Para aplicações que precisam usar PHARs, implemente uma verificação de assinatura para garantir que os arquivos não foram adulterados.
- **Sanitização de Uploads:** Não confie apenas na extensão do arquivo. Verifique o conteúdo real do arquivo (ex: usando `getimagesize` para imagens) para garantir que não é um PHAR disfarçado.

<!-- end list -->
