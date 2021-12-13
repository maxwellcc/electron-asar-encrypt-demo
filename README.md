# Aplicar código-fonte de criptografia ao Electron

Observação: este repositório ramificado parte do projeto de *toyobayashi* (https://github.com/toyobayashi/electron-asar-encrypt-demo)

Correções na tradução para Português serão bem-vindas!

## Por que existe este repositório？

Como todos sabemos, [Electron] (https: electronjs.org) não fornece oficialmente uma forma de proteger o código-fonte. Para empacotar um aplicativo Electron, para ser franco, é [copiar o código-fonte para um local fixo] (http: electronjs.orgdocstutorialapplication-distribution), como o diretório `resourcesapp` no Windows Linux. Ao executar o aplicativo Electron, Electron trata este diretório como um projeto Node.js para executar o código JS nele. Embora Electron reconheça o pacote de código no formato ASAR, ele pode empacotar todo o código-fonte em um arquivo `app.asar` e colocá-lo no diretório` resources`. Electron trata `app.asar` como uma pasta e executa o código dentro, mas ASAR Os arquivos no pacote não são criptografados, apenas juntando todos os arquivos em um arquivo e adicionando as informações do cabeçalho do arquivo. É fácil extrair todo o código-fonte do pacote ASAR usando a biblioteca oficial `asar`. Sem o efeito da criptografia, é que há um pouco mais de limite para que os iniciantes tenham acesso ao código-fonte e não há pressão para ter um pouco mais de conhecimento.

Então, eu estava pensando em como criptografar o pacote ASAR para evitar que o código-fonte comercial seja facilmente adulterado por algumas pessoas interessadas ou injetado algum código malicioso antes da distribuição. Aqui está uma maneira de completar a criptografia sem recompilar o Electron.
## Iniciando

``` bash
git clone https://github.com/maxwellcc/electron-asar-encrypt-demo.git
cd ./electron-asar-encrypt-demo
npm install 
npm start 
npm test 
```

## Encriptação

Tome AES-256-CBC como exemplo, primeiro gere a chave e salve-a em um arquivo local para facilitar a importação do script de empacotamento JS e o inlining de inclusão de C++.
``` js
// Este script não será empacotado no cliente, é para desenvolvimento local
const fs = require('fs')
const path = require('path')
const crypto = require('crypto')

fs.writeFileSync(path.join(__dirname, 'src/key.txt'), Array.prototype.map.call(crypto.randomBytes(32), (v => ('0x' + ('0' + v.toString(16)).slice(-2)))))
```

Isso irá gerar um arquivo `key.txt` em` src`, o conteúdo dentro é assim:

```
0x87,0xdb,0x34,0xc6,0x73,0xab,0xae,0xad,0x4b,0xbe,0x38,0x4b,0xf5,0xd4,0xb5,0x43,0xfe,0x65,0x1c,0xf5,0x35,0xbb,0x4a,0x78,0x0a,0x78,0x61,0x65,0x99,0x2a,0xf1,0xbb
```

Para criptografar ao empacotar, use a API do asar.createPackageWithOptions () da biblioteca asar:

``` ts
/// <reference types="node" />

declare namespace asar {
  // ...
  export function createPackageWithOptions(
    src: string,
    dest: string,
    options: {
      // ...
      transform?: (filePath: string) => NodeJS.ReadWriteStream | void;
    }
  ): Promise<void>
}

export = asar;
```

No terceiro parâmetro, passe a opção `transform`, que é uma função que retorna um fluxo` ReadWriteStream` legível e gravável para processar o arquivo e retorna `undefined` para não processar o arquivo. Esta etapa criptografa todos os arquivos JS e os insere no pacote ASAR.

``` js
// Este script não será empacotado no cliente, é para desenvolvimento local

const crypto = require('crypto')
const path = require('path')
const fs = require('fs')
const asar = require('asar')

// Leia a chave e faça um Buffer
const key = Buffer.from(fs.readFileSync(path.join(__dirname, 'src/key.txt'), 'utf8').trim().split(',').map(v => Number(v.trim())))

asar.createPackageWithOptions(
  path.join(__dirname, './app'),
  path.join(__dirname, './test/resources/app.asar'),
  {
    unpack: '*.node', // C++ 模块不打包
    transform (filename) {
      if (path.extname(filename) === '.js') {
        // Gerar um vetor de inicialização aleatório de 16 bytes IV
        const iv = crypto.randomBytes(16)

        // Você colocou o IV nos dados criptografados
        let append = false

        const cipher = crypto.createCipheriv(
          'aes-256-cbc',
          key,
          iv
        )
        cipher.setAutoPadding(true)
        cipher.setEncoding('base64')

        // Reescreva Readable.prototype.push para colocar o IV no topo dos dados criptografados
        const _p = cipher.push
        cipher.push = function (chunk, enc) {
          if (!append && chunk != null) {
            append = true
            return _p.call(this, Buffer.concat([iv, chunk]), enc)
          } else {
            return _p.call(this, chunk, enc)
          }
        }
        return cipher
      }
    }
  }
)
```

## Descriptografia do processo principal

A descriptografia é feita quando o cliente está em execução. Como o mecanismo V8 não pode executar o JS criptografado, ele deve ser descriptografado e, em seguida, lançado no V8 para ser executado. Há uma ênfase especial aqui. O código do cliente pode ser destruído por qualquer pessoa, portanto, a chave não pode ser escrita claramente e o arquivo de configuração não pode ser colocado, portanto, só pode ser inserido em C ++. Escreva um módulo nativo em C ++ para obter a descriptografia, e este módulo não pode exportar métodos de descriptografia, caso contrário, não faz sentido. Além disso, a chave não pode ser escrita como string no código-fonte C++, porque a string pode ser encontrada diretamente no arquivo binário compilado.

Que? Não adianta se você não exportar? É muito simples. Hackeie a API do Node.js, certifique-se de que está OK se não estiver disponível de fora e, em seguida, use diretamente este módulo nativo como o módulo de entrada e, em seguida, solicite a entrada real JS no módulo nativo . A seguir está a lógica JS equivalente

``` js
// Escreva a seguinte lógica em C ++ para fazer a chave ser compilada na biblioteca dinâmica
// Somente descompilando a biblioteca dinâmica ela pode ser analisada

const moduleParent = module.parent;
if (module !== process.mainModule || (moduleParent !== Module && moduleParent !== undefined && moduleParent !== null)) {
  // Se o módulo nativo não for o ponto de entrada, saia com um erro
  dialog.showErrorBox('Error', 'This program has been changed by others.')
  app.quit()
}

const { app, dialog } = require('electron')
const crypto = require('crypto')
const Module = require('module')

function getKey () {
  // Inline a chave gerada pelo script JS aqui
  // const unsigned char key[32] = {
  //   #include "key.txt"
  // };
  return KEY
}

function decrypt (body) { // body 是 Buffer
  const iv = body.slice(0, 16) // Os primeiros 16 bytes são IV
  const data = body.slice(16) // Depois de 16 bytes é o código criptografado

  // É melhor usar a biblioteca nativa para descriptografar, a API do Node corre o risco de ser interceptada

  // const clearEncoding = 'utf8' // A saída é uma string
  // const cipherEncoding = 'binary' // A entrada é binária
  // const chunks = [] // Salve a string segmentada
  // const decipher = crypto.createDecipheriv(
  //   'aes-256-cbc',
  //   getKey(),
  //   iv
  // )
  // decipher.setAutoPadding(true)
  // chunks.push(decipher.update(data, cipherEncoding, clearEncoding))
  // chunks.push(decipher.final(clearEncoding))
  // const code = chunks.join('')
  // return code

  // [native code]
}

const oldCompile = Module.prototype._compile
// Reescrever Module.prototype._compile
// Não vou escrever muito sobre o motivo, basta olhar para o código-fonte do Node.
Object.defineProperty(Module.prototype, '_compile', {
  enumerable: true,
  value: function (content, filename) {
    if (filename.indexOf('app.asar') !== -1) {
      // Se este JS estiver em app.asar, descriptografe-o primeiro
      return oldCompile.call(this, decrypt(Buffer.from(content, 'base64')), filename)
    }
    return oldCompile.call(this, content, filename)
  }
})

try {
  // O processo principal cria a janela aqui, se você precisar passar a chave para JS, é melhor não passá-la
  require('./main.js')(getKey()) 
} catch (err) {
  // Impedir que o Electron não saia após relatar um erro
  dialog.showErrorBox('Error', err.stack)
  app.quit()
}
```

Para escrever o código acima em C ++, há uma questão: Como obter a função `require` do JS em C ++?

Olhando para o código-fonte do Node, você sabe que chamar `require` é equivalente a chamar` Module.prototype.require`, então contanto que você possa obter o objeto `module`, você também pode obter a função` require`. Infelizmente, a NAPI não expôs o objeto `módulo` no retorno de chamada de inicialização do módulo. Alguém mencionou o PR. No entanto, o oficial parece considerar alguns motivos (em linha com o padrão do Módulo ES) e não quer expor o` módulo`, apenas O objeto `exports`, ao contrário do código JS no módulo Node CommonJS, é envolvido por uma camada de funções:

``` js
function (exports, require, module, __filename, __dirname) {
  // O código escrito por mim está aqui
}
```

Lendo a documentação do Node.js com atenção, você pode ver que há algo como `global.process.mainModule` no capítulo do processo, o que significa que o módulo de entrada pode ser obtido globalmente, basta atravessar o array` children` do módulo e olhe para baixo, Comparando `module.exports` etc. que não são iguais a` extensions`, você pode encontrar o objeto `module` do módulo nativo atual.

Primeiro, encapsule o método de execução do script.

``` cpp
#include <string>
#include "napi.h"

// Primeiro encapsule o método de execução do script
Napi::Value RunScript(Napi::Env& env, const Napi::String& script) {
  napi_value res;
  NAPI_THROW_IF_FAILED(env, napi_run_script(env, script, &res), env.Undefined());
  return Napi::Value(env, res); // env.RunScript(script);
}

Napi::Value RunScript(Napi::Env& env, const std::string& script) {
  return RunScript(env, Napi::String::New(env, script)); // env.RunScript(script);
}

Napi::Value RunScript(Napi::Env& env, const char* script) {
  return RunScript(env, Napi::String::New(env, script)); // env.RunScript(script);
}
```

`node-addon-api` v3 e superior podem ser usados ​​diretamente:

``` cpp
Napi::Value Napi::Env::RunScript(const char* utf8script);
Napi::Value Napi::Env::RunScript(const std::string& utf8script);
Napi::Value Napi::Env::RunScript(Napi::String script);
```

Então você pode felizmente JS em C++.

``` cpp
Napi::Value GetModuleObject(const Napi::Env& env, const Napi::Object& exports) {
  std::string script = "(function (exports) {\n"
    "function findModule(start, target) {\n"
    "  if (start.exports === target) {\n"
    "    return start;\n"
    "  }\n"
    "  for (var i = 0; i < start.children.length; i++) {\n"
    "    var res = findModule(start.children[i], target);\n"
    "    if (res) {\n"
    "      return res;\n"
    "    }\n"
    "  }\n"
    "  return null;\n"
    "}\n"
    "return findModule(process.mainModule, exports);\n"
    "});";
  Napi::Function find_function = RunScript(env, script).As<Napi::Function>();
  Napi::Value res = find_function({ exports });
  if (res.IsNull()) {
    Napi::Error::New(env, "Cannot find module object.").ThrowAsJavaScriptException();
  }
  return res;
}
Napi::Function MakeRequireFunction(const Napi::Env& env, const Napi::Object& mod) {
  std::string script = "(function makeRequireFunction(mod) {\n"
      "const Module = mod.constructor;\n"

      "function validateString (value, name) { if (typeof value !== 'string') throw new TypeError('The \"' + name + '\" argument must be of type string. Received type ' + typeof value); }\n"

      "const require = function require(path) {\n"
      "  return mod.require(path);\n"
      "};\n"

      "function resolve(request, options) {\n"
        "validateString(request, 'request');\n"
        "return Module._resolveFilename(request, mod, false, options);\n"
      "}\n"

      "require.resolve = resolve;\n"

      "function paths(request) {\n"
        "validateString(request, 'request');\n"
        "return Module._resolveLookupPaths(request, mod);\n"
      "}\n"

      "resolve.paths = paths;\n"

      "require.main = process.mainModule;\n"

      "require.extensions = Module._extensions;\n"

      "require.cache = Module._cache;\n"

      "return require;\n"
    "});";

  Napi::Function make_require = RunScript(env, script).As<Napi::Function>();
  return make_require({ mod }).As<Napi::Function>();
}
```

``` cpp
#include <unordered_map>

struct AddonData {
  // Salvar referência do módulo Node
  std::unordered_map<std::string, Napi::ObjectReference> modules;
  // Referências de função armazenada
  std::unordered_map<std::string, Napi::FunctionReference> functions;
};

Napi::Value ModulePrototypeCompile(const Napi::CallbackInfo& info) {
  AddonData* addon_data = static_cast<AddonData*>(info.Data());
  Napi::Function old_compile = addon_data->functions["Module.prototype._compile"].Value();
  // Recomenda-se o uso da biblioteca C/C++ para descriptografia
  // ...
}

Napi::Object Init(Napi::Env env, Napi::Object exports) {
  Napi::Object this_module = GetModuleObject(env, exports).As<Napi::Object>();
  Napi::Function require = MakeRequireFunction(env, this_module);
  // const mainModule = process.mainModule
  Napi::Object main_module = env.Global().As<Napi::Object>().Get("process").As<Napi::Object>().Get("mainModule").As<Napi::Object>();
  // const electron = require('electron')
  Napi::Object electron = require({ Napi::String::New(env, "electron") }).As<Napi::Object>();
  // require('module')
  Napi::Object module_constructor = require({ Napi::String::New(env, "module") }).As<Napi::Object>();
  // module.parent
  Napi::Value module_parent = this_module.Get("parent");

  if (this_module != main_module || (module_parent != module_constructor && module_parent != env.Undefined() && module_parent != env.Null())) {
    // Recomenda-se o uso da biblioteca CC ++ para descriptografia
    // Sair após aviso pop-up
  }

  AddonData* addon_data = env.GetInstanceData<AddonData>();

  if (addon_data == nullptr) {
    addon_data = new AddonData();
    env.SetInstanceData(addon_data);
  }

  // require('crypto')
  // addon_data->modules["crypto"] = Napi::Persistent(require({ Napi::String::New(env, "crypto") }).As<Napi::Object>());

  Napi::Object module_prototype = module_constructor.Get("prototype").As<Napi::Object>();
  addon_data->functions["Module.prototype._compile"] = Napi::Persistent(module_prototype.Get("_compile").As<Napi::Function>());
  module_prototype["_compile"] = Napi::Function::New(env, ModulePrototypeCompile, "_compile", addon_data);

  try {
    require({ Napi::String::New(env, "./main.js") }).Call({ getKey() });
  } catch (const Napi::Error& e) {
    // Sair após pop-up
    // ...
  }
  return exports;
}

// Sem ponto e vírgula, NODE_API_MODULE é uma macro
NODE_API_MODULE(NODE_GYP_MODULE_NAME, Init)
```

Quando vejo isso, posso perguntar por que tenho que escrever JS em C ++ por muito tempo. Não é possível usar `RunScript ()`? Conforme mencionado anteriormente, o runScript diretamente precisa escrever JS como uma string, que existe como está no arquivo binário compilado, e a chave vazará. Escrever essa lógica em C ++ pode aumentar a dificuldade de reversão

O resumo é assim:

1. `main.node` (Compilado) Dentro de require `main.js` (criptografado)
2. `main.js` （Criptografado) dentro, em seguida, requer outro JS criptografado, crie janelas, etc.

Em particular, a entrada deve ser main.node. Se não for, é muito provável que o invasor hackeará a API do Node no JS antes de main.node e fará com que a chave vaze. Por exemplo, um arquivo de entrada:

``` js
const crypto = require('crypto')

const old = crypto.createDecipheriv
crypto.createDecipheriv = function (...args) {
  console.log(...args) // 密钥被输出
  return old.call(crypto, ...args)
}

const Module = require('module')

const oldCompile = Module.prototype._compile
      
Module.prototype._compile = function (content, filename) {
  console.log(content) // JS 源码被输出
  return oldCompile.call(this, content, filename)
}

process.argv.length = 1

require('./main.node')
// Ou Module._load('./main.node', module, true)
```

Além disso, a depuração do Node.js deve ser desabilitada no JS do processo principal, caso contrário, o código pode ser visto nas Ferramentas do desenvolvedor do Chrome.

``` js
for (let i = 0; i < process.argv.length; i++) {
  if (process.argv[i].startsWith('--inspect') || process.argv[i].startsWith('--remote-debugging-port')) {
    throw new Error('Not allow debugging this program.')
  }
}
```

Mas este método não pode impedir a configuração de `process.argv.length = 1` no script carregado antes de main.node, então a chave é evitar que o arquivo de entrada seja alterado para outros scripts JS.

## Descriptografia do processo de renderização

Semelhante à lógica do processo principal, macros predefinidas podem ser usadas para distinguir entre o processo principal e o processo de renderização em C ++. Compile um `renderer.node` para o processo de renderização. O módulo nativo carregado pelo processo de renderização deve ser um `módulo sensível ao contexto`. O módulo escrito com NAPI já é sensível ao contexto, então não há problema. Se você usar a API V8 para escrevê-lo, não funcionará.

Há uma limitação aqui. Você não pode referenciar diretamente a tag `<script>` em HTML para carregar JS, porque o `<script>` em HTML não vai para `Module.prototype._compile`, então` browserWindow só pode ser chamado no processo principal. webContents.executeJavaScript () `para carregar o módulo nativo para cada janela primeiro e, em seguida, requerer outros arquivos JS que podem precisar ser descriptografados.

## Limitações

* A opção `nodeIntegration` deve ser ativada. Também não pode usar `preload` para pré-carregar scripts, porque o módulo nativo não encontrará sua própria instância de` módulo`, então `require` não pode ser usado
* Só pode criptografar JS, não outros tipos de arquivos, como JSON, recursos de imagem etc.
* Todos os métodos de carregamento de JS que não usam `Module.prototype._compile` não podem carregar JS criptografado. Por exemplo, métodos de carregamento de script que dependem de tags HTML` <script> `falham e Webpack dynamic import` import () `falha.
* Se houver muitos arquivos JS, o impacto no desempenho causado pela descriptografia será maior. A seguir, falaremos sobre como reduzir o JS que precisa ser criptografado
* Não pode ser implementado em JS puro, C ++ deve ser usado para compilar a chave da chave e o método de descriptografia
* Não pode ser feito em um aplicativo pago
* Não é absolutamente seguro. Módulos nativos descompilados ainda apresentam o risco de vazamentos de chaves e métodos de criptografia sendo conhecidos, mas em comparação com o empacotamento ASAR puro, o limite para cracking é ligeiramente aumentado e o código-fonte não é tão fácil de ser acessado. Se alguém realmente deseja destruir seu código, esta abordagem pode não ser suficiente para defesa

maneira mais eficaz é alterar o código-fonte do Electron e recompilar o Electron. No entanto, o limite para mover a tecnologia do código-fonte é alto, e recompilar o Electron requer ciência, e a compilação é muito lenta.

## Reduza o JS que precisa ser criptografado

Existem muitos JS em `node_modules` e não precisam ser criptografados, então você pode extrair um` node_modules.asar` separado, o JS dentro não é criptografado. Mas isso trará mais oportunidades para a engenharia reversa. Outros podem injetar o código JS que desejam executar nesses pacotes NPM, o que é arriscado.

Como fazer `require` encontrar as bibliotecas dentro de` node_modules.asar`? A resposta também é hackear a API do Node.

``` js
const path = require('path')
const Module = require('module')

const originalResolveLookupPaths = Module._resolveLookupPaths

Module._resolveLookupPaths = originalResolveLookupPaths.length === 2 ? function (request, parent) {
  // Node v12+
  const result = originalResolveLookupPaths.call(this, request, parent)

  if (!result) return result

  for (let i = 0; i < result.length; i++) {
    if (path.basename(result[i]) === 'node_modules') {
      result.splice(i + 1, 0, result[i] + '.asar')
      i++
    }
  }

  return result
} : function (request, parent, newReturn) {
  // Node v10-
  const result = originalResolveLookupPaths.call(this, request, parent, newReturn)

  const paths = newReturn ? result : result[1]
  for (let i = 0; i < paths.length; i++) {
    if (path.basename(paths[i]) === 'node_modules') {
      paths.splice(i + 1, 0, paths[i] + '.asar')
      i++
    }
  }

  return result
}
```

Desta forma, está OK rotular `node_modules` como` node_modules.asar` e colocá-lo na pasta `resources` e no mesmo nível que` app.asar`.

Lembre-se de descompactar o módulo nativo `.node`

## Resumo

Criptografia durante o empacotamento e descriptografia durante o tempo de execução. A lógica de descriptografia é colocada em C ++ e deve ser carregada primeiro.

A última chave é não `console.log` no código pré-carregado e não se esqueça de desligar` devTools` e abrir `nodeIntegration` no ambiente de produção:

``` js
new BrowserWindow({
  // ...
  webPreferences: {
    nodeIntegration: true, // 渲染进程要使用 require
    contextIsolation: false, // Electron 12 开始默认值为 true，要关掉
    devTools: false // 关掉开发者工具，因为开发者工具可以看到渲染进程的代码
  }
})
```
