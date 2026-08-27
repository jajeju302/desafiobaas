## BUG #01 — [ Login Silencia Erros ]

### O que estava acontecendo
[No login, se a pessoa errasse o e-mail ou a senha, não aparecia nenhum aviso na tela. Simplesmente não acontecia nada]

### Por que acontecia
[o catch tava vazio. Ou seja, quando o Firebase soltava um erro (senha errada ou usuário não encontrado), o código ignorava o erro.]

### Como corrigi
[Antes:

catch {
  // catch vazio — erro engolido
}

Depois:

catch (err) {
  const msg = err instanceof Error ? err.message : "Erro desconhecido";
  if (msg.includes("invalid-credential") || msg.includes("wrong-password")) {
    setErro("E-mail ou senha incorretos.");
  } else if (msg.includes("user-not-found")) {
    setErro("Nenhuma conta encontrada com este e-mail.");
  } else {
    setErro("Erro ao entrar. Tente novamente.");
  }
  }]

## BUG #02 — [ Middleware com Condição Invertida]

### O que estava acontecendo
[O sistema tava fazendo o oposto do que devia: quem já tava logado era jogado pra tela de login, e quem NÃO tava logado conseguia entrar em rotas que deveriam ser bloqueadas.]

### Por que acontecia
[No middleware, a checagem estava: if (token) redireciona pro login. Só que isso tá invertido, o certo era redirecionar quando NÃO existe token (!token). Ou seja, a regra de proteção das rotas tava de cabeça pra baixo.]

### Como corrigi
[Antes:
if (token) {
  return NextResponse.redirect(new URL("/login", request.url));
}

Depois:
if (!token) {
  return NextResponse.redirect(new URL("/login", request.url));
}]

## BUG #03 — [Confirmação de Senha Compara com Nome]

### O que estava acontecendo
[Na hora do cadastro, o campo "confirmar senha" não validava direito: em vez de comparar com a senha digitada, o código comparava com o nome do usuário.]

### Por que acontecia
[Foi um erro de nome de variável parecido, usaram nome no lugar de confirmarSenha:
if (senha !== nome) ]

### Como corrigi
[Antes:
if (senha !== nome) 

Depois:
if (senha !== confirmarSenha) ]

## BUG #04 — [Query Sem Filtro de userId]

### O que estava acontecendo
[Todo mundo via os personagens de todo mundo. Um usuário conseguia enxergar os personagens de outra pessoa.]

### Por que acontecia
[A busca no Firestore trazia TODOS os documentos da coleção, sem nenhum filtro pra restringir só aos personagens daquele usuário logado:
const q = query(collection(db, "personagens")); ]

### Como corrigi
[Antes:
const q = query(collection(db, "personagens"));

Depois:
import { where } from "firebase/firestore";
const q = query(
  collection(db, "personagens"),
  where("userId", "==", uid)
); ]

## BUG #05 — [Nome de Coleção Errado no Create]

### O que estava acontecendo
[Quando criava um personagem novo, ele simplesmente sumia, parecia que o cadastro tinha falhado, mas ele só não aparecia na lista.]

### Por que acontecia
[O código salvava o personagem numa coleção chamada "personagem" (no singular), mas o resto do sistema lê e escreve em "personagens" (no plural). Então o personagem novo ia parar num lugar que ninguém consultava:
const ref = await addDoc(collection(db, "personagem"), { ... });]

### Como corrigi
[Antes:
const ref = await addDoc(collection(db, "personagem"), { ... });

Depois:
const ref = await addDoc(collection(db, "personagens"), { ... });]

## BUG #06 — [setDoc Apaga o Documento Inteiro]

### O que estava acontecendo
[Ao equipar um item, o personagem perdia TODOS os outros dados (nome, atributos, outros itens), só sobrava o item que acabou de ser equipado.
]

### Por que acontecia
[A função usava setDoc, que substitui o documento inteiro pelo que foi passado, quando deveria usar updateDoc, que só atualiza os campos informados sem mexer no resto:
await setDoc(doc(db, "personagens", personagemId), { [slot]: itemId });]

### Como corrigi
[Antes:
await setDoc(doc(db, "personagens", personagemId), { [slot]: itemId });

Depois:
await updateDoc(doc(db, "personagens", personagemId), { [slot]: itemId });
]

## BUG #07 — [Deletar Usa Índice Como ID]

### O que estava acontecendo
[Ao apagar um personagem da lista, às vezes apagava o errado, geralmente outro que nem tinha sido clicado.
]

### Por que acontecia
[O código usava a posição do item na lista (tipo 0, 1, 2...) como se fosse o ID do documento no Firestore. Só que posição no array e ID do documento são coisas totalmente diferentes, então acabava apagando o documento que por acaso tinha aquele número como ID, ou nem apagava nada:
await deleteDoc(doc(db, "personagens", String(indice)));
]

### Como corrigi
[Antes:
await deleteDoc(doc(db, "personagens", String(indice)));

Depois:
await deleteDoc(doc(db, "personagens", personagem.id));
]

## BUG #08 — [Security Rules Abertas]

### O que estava acontecendo
[ualquer pessoa, nem precisava estar logada, conseguia ler, criar, editar ou apagar qualquer dado do banco direto, só sabendo a URL do projeto no Firebase.
]

### Por que acontecia
[As regras de segurança do Firestore estavam liberando leitura e escrita pra qualquer requisição, sem checar login nem dono do dado:
match /{document=**} 
  allow read, write: if true;
]

### Como corrigi
[Antes:
match /{document=**} {
  allow read, write: if true;
}

Depois:
match /personagens/{personagemId} {
  allow read: if request.auth != null &&
              request.auth.uid == resource.data.userId;
  allow create: if request.auth != null &&
                request.auth.uid == request.resource.data.userId;
  allow update, delete: if request.auth != null &&
                        request.auth.uid == resource.data.userId;
}
]
