---
title: Como o React Diferencia uma Classe de uma Função?
date: '2018-12-02'
spoiler: Falamos sobre classes, new, instanceof, cadeias de prototype e design de API.
---

Considere esse componente `Greeting` que é definido como uma função:

```jsx
function Greeting() {
  return <p>Hello</p>;
}
```

O React também dá suporte para definirmos ele como uma classe:

```jsx
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}
```

(Até [recentemente](https://reactjs.org/docs/hooks-intro.html)), essa era a única forma de usar funcionalidades como `state`

Quando queremos renderizar `<Greeting />` você não se importa como ele é definido:

```jsx
// Classe ou função — tanto faz.
<Greeting />
```

Mas o *React* se importa com a diferença!

Se `Greeting` é uma função, o React precisa invocar ela:

```jsx
// Seu código
function Greeting() {
  return <p>Hello</p>;
}

// Dentro do React
const result = Greeting(props); // <p>Hello</p>
```

Mas se `Greeting` for uma classe, o React precisa instanciar ela com o operador `new` e *depois* invocar o método `render` na instância que foi criada:

```jsx
// Seu código
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

// Dentro do React
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

Em ambos os casos o objetivo do React é renderizar o nó (nesse exemplo, `<p>Hello</p>`). Mas os passos exatos dependem em como `Greeting` foi definido.

**Então como o React sabe se algo é uma classe ou uma função?**

Assim como meu [artigo anterior](/why-do-we-write-super-props/), **você não *precisa* saber sobre isso para ser produtivo com o React.** Eu não sabia isso durante anos. Por favor não transforme isso numa questão de entrevista. Na verdade esse artigo é mais sobre JavaScript do que sobre React.

Esse blog é para o leitor curioso que deseja saber o *porquê* de o React trabalhar de certa forma. Você é essa pessoa? Então vamos mergulhar juntos.

**Essa é uma longa jornada. Fique firme. Esse artigo não possui muitas informações sobre React, mas mas iremos verificar alguns aspectos sobre `new`, `this`, `class`, arrow functions, `prototype`, `__proto__`, `instanceof` e como essas coisas trabalham juntas em JavaScript. Felizmente você não precisa pensar muito sobre isso quando você *utiliza* o React. Mas se você está implementando o React...**

(Se você quer realmente apenas saber a resposta, pule para o final.)

----

Primeiro, precisamos entender poruqe é importante tratar funções e classes de forma diferente. Veja como usamos o operador `new` quando chamamos uma classe:

```jsx{5}
// Se Greeting é uma função
const result = Greeting(props); // <p>Hello</p>

// Se Greeting é uma classe
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

Vamos ver o que o operador `new` faz no JavaScript.

---

Nos velhos tempos, o JavaScript não possuía classes. Contudo, você poderia criar um padrão semelhante às classes usando apenas funções. **Concretamente, você pode usar *qualquer* função de forma parecida com um construtor de uma classe adicionando o `new` antes de sua chamada:**

```jsx
// Apenas uma função
function Person(name) {
  this.name = name;
}

var fred = new Person('Fred'); // ✅ Person {name: 'Fred'}
var george = Person('George'); // 🔴 Não irá funcionar
```

Você ainda pode escrever seu código assim hoje! Experimente nas DevTools.

Se você chamou `Person('Fred')` **sem** o `new`, o `this` dentro dela iria apontar para algo global e seria inútil (por exemplo, `window` ou `undefined`). Então nosso código iria quebrar ou fazer algo estranho como `window.name`.

Adicionando o `new` antes da chamada, nós dizemos: "Hey JavaScript, eu sei que `Person` é apenas uma função, mas vamos fingir que ela é algo parecido com um método construtor de uma classe. **Crie um objeto `{}` e aponte o `this` dentro da função `Person` para esse objeto, dessa forma posso atribuir algo como `this.name`. Então retorne esse objeto para mim.**"

Isso é o que o operador `new` faz.

```jsx
var fred = new Person('Fred'); // Mesmo objeto `this` dentro de `Person`
```

O operador `new` também faz com que tudo que pusermos em `Person.prototype` esteja disponível no objeto `fred`:

```jsx{4-6,9}
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name);
}

var fred = new Person('Fred');
fred.sayHi();
```

Isso era a forma como as pessoas simulavam classes em JavaScript antes de serem adicionadas diretamente.

---

O `new` já existe por um bom tempo no JavaScript. Porém, classes são mais recente. Elas nos permitem reescrever o código acima para corresponder às nossas intenções de forma mais direta:

```jsx
class Person {
  constructor(name) {
    this.name = name;
  }
  sayHi() {
    alert('Hi, I am ' + this.name);
  }
}

let fred = new Person('Fred');
fred.sayHi();
```

*Captar as intenções do desenvolvedor* é algo importante no design de APIs e linguagens.

Se você escrever uma função, o JavaScript não sabe dizer se ela deve ser chamada como `alert()` ou se ela serve como um construtor como `new Person()`. Esquecer de especificar o `new` para uma função como `Person` iria gerar um comportamento confuso.

**A sintaxe das Classes nos permitem dizer: "Isso não é apenas uma função - é uma classe e ela tem um construtor".** Se você esquecer o `new` ao chamar ela, o JavaScript irá emitir um erro:

```jsx
let fred = new Person('Fred');
// ✅  Se Person é uma função: funciona sem problemas
// ✅  Se Person é uma classe: funciona sem problemas também

let george = Person('George'); // We forgot `new`
// 😳 Se Person é uma função como um construtor: comportamento confuso
// 🔴 Se Person é uma classe: falha imediatamente
```

Isso nos ajuda a pegar erros antecipadamente ao invés de esperar algum erro obscuro como `this.name` sendo tratado como `window.name` ao invés de `george.name`.

Então isso quer dizer que o React deve colocar o operador `new` antes de chamar uma classe. Ele não pode apenas chamar como uma função comum, pois o JavaScript iria tratar como um erro!

```jsx
class Counter extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

// 🔴 O React não pode fazer assim
const instance = Counter(props);
```

Isso traria problemas.

---

Antes de vermos como o React resolve isso, é importante lembrar que a maioria das pessoas que utilizam o React usam compiladores como o Babel para compilar funcionalidades modernas como classes para navegadores mais antigos. Então devemos considerar esses compiladores em nosso design.

Nas primeiras versões do Babel, classes poderiam ser chamadas sem o `new`. Porém, isso foi resolvido - criando um pouco de código extra:

```jsx
function Person(name) {
  // Um resultado um pouco simplificado do Babel:
  if (!(this instanceof Person)) {
    throw new TypeError("Cannot call a class as a function");
  }
  // Nosso código:
  this.name = name;
}

new Person('Fred'); // ✅ Ok
Person('George');   // 🔴 Não pode chamar uma classe como uma função
``` 

Você deve ter visto código parecido com esse em seu `bundle`. Isso é o que todas aquelas funções `_classCallCheck` fazem. (Você pode reduzir o tamanho do seu `bundle` ao optar por um "modo livre" sem nenhuma verificação, mas isso pode complicar sua transição eventual para classes nativas.)

---

Agora você deve entender um pouco da diferença entre chamar algo com ou sem o operador `new`:

|  | `new Person()` | `Person()` |
|---|---|---|
| `classe` | ✅ `this` é uma instância de `Person` | 🔴 `TypeError`
| `função` | ✅ `this` é uma instância de `Person` | 😳 `this` é `window` ou `undefined` |

Esse é o motivo pelo qual o React deve chamar seu componente corretamente. **Se seu componente é definido como uma classe, o React precisa usar o `new` ao chamar ele.**

Então o React pode apenas checar se algo é uma classe ou não?

Não é tão fácil! Mesmo que pudéssemos [diferenciar uma classe de uma função em JavaScript](https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function), isso ainda não iria funcionar para classes processadas por ferramentas como o Babel. Para o navegador, elas são apenas funções. Azar para o React.

---

Certo, então talvez o React poderia apenas usar o `new` em todas as chamadas? Infelizmente isso não iria sempre funcionar.

Com funções comuns, chamar elas com o operador `new` iria dar a elas um objeto de instância como `this`. Isso é o desejado para funções escritas como construtores (como nossa `Person` acima), mas seria confuso para componentes de função:

```jsx
function Greeting() {
  // Não esperamos que `this` seja nenhum tipo de instância aqui
  return <p>Hello</p>;
}
```

Isso pode ser tolerável. Mas há duas *outras* razões que matam essa ideia.

---

The first reason why always using `new` wouldn’t work is that for native arrow functions (not the ones compiled by Babel), calling with `new` throws an error:

```jsx
const Greeting = () => <p>Hello</p>;
new Greeting(); // 🔴 Greeting is not a constructor
```

This behavior is intentional and follows from the design of arrow functions. One of the main perks of arrow functions is that they *don’t* have their own `this` value — instead, `this` is resolved from the closest regular function:

```jsx{2,6,7}
class Friends extends React.Component {
  render() {
    const friends = this.props.friends;
    return friends.map(friend =>
      <Friend
        // `this` is resolved from the `render` method
        size={this.props.size}
        name={friend.name}
        key={friend.id}
      />
    );
  }
}
```

Okay, so **arrow functions don’t have their own `this`.** But that means they would be entirely useless as constructors!

```jsx
const Person = (name) => {
  // 🔴 This wouldn’t make sense!
  this.name = name;
}
```

Therefore, **JavaScript disallows calling an arrow function with `new`.** If you do it, you probably made a mistake anyway, and it’s best to tell you early. This is similar to how JavaScript doesn’t let you call a class *without* `new`.

This is nice but it also foils our plan. React can’t just call `new` on everything because it would break arrow functions! We could try detecting arrow functions specifically by their lack of `prototype`, and not `new` just them:

```jsx
(() => {}).prototype // undefined
(function() {}).prototype // {constructor: f}
```

But this [wouldn’t work](https://github.com/facebook/react/issues/4599#issuecomment-136562930) for functions compiled with Babel. This might not be a big deal, but there is another reason that makes this approach a dead end.

---

Another reason we can’t always use `new` is that it would preclude React from supporting components that return strings or other primitive types.

```jsx
function Greeting() {
  return 'Hello';
}

Greeting(); // ✅ 'Hello'
new Greeting(); // 😳 Greeting {}
```

This, again, has to do with the quirks of the [`new` operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new) design. As we saw earlier, `new` tells the JavaScript engine to create an object, make that object `this` inside the function, and later give us that object as a result of `new`.

However, JavaScript also allows a function called with `new` to *override* the return value of `new` by returning some other object. Presumably, this was considered useful for patterns like pooling where we want to reuse instances:

```jsx{1-2,7-8,17-18}
// Created lazily
var zeroVector = null;

function Vector(x, y) {
  if (x === 0 && y === 0) {
    if (zeroVector !== null) {
      // Reuse the same instance
      return zeroVector;
    }
    zeroVector = this;
  }
  this.x = x;
  this.y = y;
}

var a = new Vector(1, 1);
var b = new Vector(0, 0);
var c = new Vector(0, 0); // 😲 b === c
```

However, `new` also *completely ignores* a function’s return value if it’s *not* an object. If you return a string or a number, it’s like there was no `return` at all.

```jsx
function Answer() {
  return 42;
}

Answer(); // ✅ 42
new Answer(); // 😳 Answer {}
```

There is just no way to read a primitive return value (like a number or a string) from a function when calling it with `new`. So if React always used `new`, it would be unable to add support components that return strings!

That’s unacceptable so we need to compromise.

---

What did we learn so far? React needs to call classes (including Babel output) *with* `new` but it needs to call regular functions or arrow functions (including Babel output) *without* `new`. And there is no reliable way to distinguish them.

**If we can’t solve a general problem, can we solve a more specific one?**

When you define a component as a class, you’ll likely want to extend `React.Component` for built-in methods like `this.setState()`. **Rather than try to detect all classes, can we detect only `React.Component` descendants?**

Spoiler: this is exactly what React does.

---

Perhaps, the idiomatic way to check if `Greeting` is a React component class is by testing if `Greeting.prototype instanceof React.Component`:

```jsx
class A {}
class B extends A {}

console.log(B.prototype instanceof A); // true
```

I know what you’re thinking. What just happened here?! To answer this, we need to understand JavaScript prototypes.

You might be familiar with the “prototype chain”. Every object in JavaScript might have a “prototype”. When we write `fred.sayHi()` but `fred` object has no `sayHi` property, we look for `sayHi` property on `fred`’s prototype. If we don’t find it there, we look at the next prototype in the chain — `fred`’s prototype’s prototype. And so on.

**Confusingly, the `prototype` property of a class or a function _does not_ point to the prototype of that value.** I’m not kidding.

```jsx
function Person() {}

console.log(Person.prototype); // 🤪 Not Person's prototype
console.log(Person.__proto__); // 😳 Person's prototype
```

So the “prototype chain” is more like `__proto__.__proto__.__proto__` than `prototype.prototype.prototype`. This took me years to get.

What’s the `prototype` property on a function or a class, then? **It’s the `__proto__` given to all objects `new`ed with that class or a function!**

```jsx{8}
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name);
}

var fred = new Person('Fred'); // Sets `fred.__proto__` to `Person.prototype`
```

And that `__proto__` chain is how JavaScript looks up properties:

```jsx
fred.sayHi();
// 1. Does fred have a sayHi property? No.
// 2. Does fred.__proto__ have a sayHi property? Yes. Call it!

fred.toString();
// 1. Does fred have a toString property? No.
// 2. Does fred.__proto__ have a toString property? No.
// 3. Does fred.__proto__.__proto__ have a toString property? Yes. Call it!
```

In practice, you should almost never need to touch `__proto__` from the code directly unless you’re debugging something related to the prototype chain. If you want to make stuff available on `fred.__proto__`, you’re supposed to put it on `Person.prototype`. At least that’s how it was originally designed.

The `__proto__` property wasn’t even supposed to be exposed by browsers at first because the prototype chain was considered an internal concept. But some browsers added `__proto__` and eventually it was begrudgingly standardized (but deprecated in favor of `Object.getPrototypeOf()`).

**And yet I still find it very confusing that a property called `prototype` does not give you a value’s prototype** (for example, `fred.prototype` is undefined because `fred` is not a function). Personally, I think this is the biggest reason even experienced developers tend to misunderstand JavaScript prototypes.

---

This is a long post, eh? I’d say we’re 80% there. Hang on.

We know that when we say `obj.foo`, JavaScript actually looks for `foo` in `obj`, `obj.__proto__`, `obj.__proto__.__proto__`, and so on.

With classes, you’re not exposed directly to this mechanism, but `extends` also works on top of the good old prototype chain. That’s how our React class instance gets access to methods like `setState`:

```jsx{1,9,13}
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

let c = new Greeting();
console.log(c.__proto__); // Greeting.prototype
console.log(c.__proto__.__proto__); // React.Component.prototype
console.log(c.__proto__.__proto__.__proto__); // Object.prototype

c.render();      // Found on c.__proto__ (Greeting.prototype)
c.setState();    // Found on c.__proto__.__proto__ (React.Component.prototype)
c.toString();    // Found on c.__proto__.__proto__.__proto__ (Object.prototype)
```

In other words, **when you use classes, an instance’s `__proto__` chain “mirrors” the class hierarchy:**

```jsx
// `extends` chain
Greeting
  → React.Component
    → Object (implicitly)

// `__proto__` chain
new Greeting()
  → Greeting.prototype
    → React.Component.prototype
      → Object.prototype
```

2 Chainz.

---

Since the `__proto__` chain mirrors the class hierarchy, we can check whether a `Greeting` extends `React.Component` by starting with `Greeting.prototype`, and then following down its `__proto__` chain:

```jsx{3,4}
// `__proto__` chain
new Greeting()
  → Greeting.prototype // 🕵️ We start here
    → React.Component.prototype // ✅ Found it!
      → Object.prototype
```

Conveniently, `x instanceof Y` does exactly this kind of search. It follows the `x.__proto__` chain looking for `Y.prototype` there.

Normally, it’s used to determine whether something is an instance of a class:

```jsx
let greeting = new Greeting();

console.log(greeting instanceof Greeting); // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype (✅ Found it!)
//     .__proto__ → React.Component.prototype 
//       .__proto__ → Object.prototype

console.log(greeting instanceof React.Component); // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype (✅ Found it!)
//       .__proto__ → Object.prototype

console.log(greeting instanceof Object); // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype
//       .__proto__ → Object.prototype (✅ Found it!)

console.log(greeting instanceof Banana); // false
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype 
//       .__proto__ → Object.prototype (🙅‍ Did not find it!)
```

But it would work just as fine to determine if a class extends another class:

```jsx
console.log(Greeting.prototype instanceof React.Component);
// greeting
//   .__proto__ → Greeting.prototype (🕵️‍ We start here)
//     .__proto__ → React.Component.prototype (✅ Found it!)
//       .__proto__ → Object.prototype
```

And that check is how we could determine if something is a React component class or a regular function.

---

That’s not what React does though. 😳

One caveat to the `instanceof` solution is that it doesn’t work when there are multiple copies of React on the page, and the component we’re checking inherits from *another* React copy’s `React.Component`. Mixing multiple copies of React in a single project is bad for several reasons but historically we’ve tried to avoid issues when possible. (With Hooks, we [might need to](https://github.com/facebook/react/issues/13991) force deduplication though.)

One other possible heuristic could be to check for presence of a `render` method on the prototype. However, at the time it [wasn’t clear](https://github.com/facebook/react/issues/4599#issuecomment-129714112) how the component API would evolve. Every check has a cost so we wouldn’t want to add more than one. This would also not work if `render` was defined as an instance method, such as with the class property syntax.

So instead, React [added](https://github.com/facebook/react/pull/4663) a special flag to the base component. React checks for the presence of that flag, and that’s how it knows whether something is a React component class or not.

Originally the flag was on the base `React.Component` class itself:

```jsx
// Inside React
class Component {}
Component.isReactClass = {};

// We can check it like this
class Greeting extends Component {}
console.log(Greeting.isReactClass); // ✅ Yes
```

However, some class implementations we wanted to target [did not](https://github.com/scala-js/scala-js/issues/1900) copy static properties (or set the non-standard `__proto__`), so the flag was getting lost.

This is why React [moved](https://github.com/facebook/react/pull/5021) this flag to `React.Component.prototype`: 

```jsx
// Inside React
class Component {}
Component.prototype.isReactComponent = {};

// We can check it like this
class Greeting extends Component {}
console.log(Greeting.prototype.isReactComponent); // ✅ Yes
```

**And this is literally all there is to it.**

You might be wondering why it’s an object and not just a boolean. It doesn’t matter much in practice but early versions of Jest (before Jest was Good™️) had automocking turned on by default. The generated mocks omitted primitive properties, [breaking the check](https://github.com/facebook/react/pull/4663#issuecomment-136533373). Thanks, Jest.

The `isReactComponent` check is [used in React](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L297-L300) to this day.

If you don’t extend `React.Component`, React won’t find `isReactComponent` on the prototype, and won’t treat component as a class. Now you know why [the most upvoted answer](https://stackoverflow.com/a/42680526/458193) for `Cannot call a class as a function` error is to add `extends React.Component`. Finally, a [warning was added](https://github.com/facebook/react/pull/11168) that warns when `prototype.render` exists but `prototype.isReactComponent` doesn’t.

---

You might say this story is a bit of a bait-and-switch. **The actual solution is really simple, but I went on a huge tangent to explain *why* React ended up with this solution, and what the alternatives were.**

In my experience, that’s often the case with library APIs. For an API to be simple to use, you often need to consider the language semantics (possibly, for several languages, including future directions), runtime performance, ergonomics with and without compile-time steps, the state of the ecosystem and packaging solutions, early warnings, and many other things. The end result might not always be the most elegant, but it must be practical.

**If the final API is successful, _its users_ never have to think about this process.** Instead they can focus on creating apps.

But if you’re also curious... it’s nice to know how it works.
