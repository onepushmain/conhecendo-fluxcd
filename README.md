# GitOps com FluxCD!

Os passos abaixo especificam como utilizar o FluxCD para funcionar como seu operador de GitOps. O Flux permite uma instalação e configuração bem simples, e com apenas alguns passos já conseguimos fazer a sincronização do repositório com o cluster.  

## 0.1 Pré-Requisitos:

- Um Cluster Kubernetes em uma versão **MAIOR que a 1.19** (nesse caso, estou usando o [minikube](https://minikube.sigs.k8s.io/), mas poderia ser qualquer um, desde que o mesmo tenha acesso a internet)
- Para este mini tutorial, vou usar minha conta no GitHub com um access token configurado. Porém, vale ressaltar que o Flux aceita diversos tipos de Sources, e eles serão comentados mais a frente neste doc. 

## 1: Instalação do FluxCLI

Nas versões mais recentes do FluxCD, o FluxCLI substitui o antigo FluxCTL como nosso "controller" do Flux. Assim, podemos executar comandos imperativos e instalarmos/configurarmos o FluxCD rapidamente.

Utilizando Homebrew:
`# brew install fluxcd/tap/flux`

Utilizando um ShellScript de instalação no Bash:
`bash curl -s https://fluxcd.io/install.sh | sudo bash` 

Utilizando Docker:
`# docker run -it --entrypoint=sh -v ~/.kube/config:/kubeconfig ghcr.io/fluxcd/flux-cli:v0.17.0`


## 2 Criar um Token no Github

Quando executarmos o comando de instalação, o Flux precisa ter permissão no seu repositório para ler e alterar os arquivos. Este repositório inicial será o repositório do "flux", onde suas respectivas configurações ficaram armazenadas, como por exemplo: Manifestos gerais como o *HelmRelease* ou de Controllers específicos do Flux, como os *GitRepositories*, *Kustomizations*, etc.

Caso tenha duvidas, é possível seguir o tutorial de como criar seu token no Github [aqui]( https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

## 3 Instalação Bootstrap do FluxCD

O que é uma instalação Bootstrap? 
Segundo a documentação, é chamada Bootstrap a forma de se instalar o FluxCD da maneira "GitOps de se fazer". 
Quando o FluxCD é instalado via FluxCLI, ele cria um repositório com dois objetos importantissimos:

  1- Um  **GitRepository**, apontando para o repositório
  2 - Um arquivo do tipo **Kustomization**, definindo alguns padrões para a 
sincronização.

Por causa desses dois objetos, o próprio flux consegue se sincronizar e reconciliar automaticamente, de uma forma totalmente GitOps.

Lembrando também que essa forma não é a única para a instalação do Flux de forma Bootstrap. Também é possível utilizar o [Terraform](https://github.com/fluxcd/terraform-provider-flux).

## 4 Executando a instalação:

Primeiramente, faça export do seu token e seu usuário do GitHub, desta forma:

```
export GITHUB_TOKEN=<seu_token>
export GITHUB_USER=<seu_usuario>
```

Após exportar as variaveis que iremos utilizar, vamos executar a instalacao:

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=<nome-do-repo> \
  --branch=main \
  --path=./meu-cluster \
  --personal
```
Todas as opções do comando são bem auto-explicativas, menos a **personal**. 
Essa opção define se quem está rodando o comando é uma organização ou um usuário do git. Nesse caso, como estou rodando do meu usuário (onepushmain), preciso defini-la. 

Prontinho! Após isso, se não houver erros, o FluxCD está instalado com sucesso!
A instalação bootstrap faz o seguinte:
- Cria um repositório com o nome que foi indicado no campo --repository
- Adiciona todos os componentes necessários para o Flux funcionar
- Faz o deploy desses mesmos componentes no seu cluster
- Configura um listener no --path definido. Assim, quando houver qualquer mudança nesse caminho, ele irá atualizar no cluster.




##  5 Adicionar um repositório a ser sincronizado

Agora que temos o FluxCD instalado, precisamos configurar nossa [Source](https://fluxcd.io/docs/concepts/#sources). Para configurarmos, precisamos de um objeto de [GitRepository](https://fluxcd.io/docs/components/source/gitrepositories/) que servira como listener do repositório, e ficará observando o repositório por um intervalo definido, até que hajam mudanças.

Existem outros tipos de Source, como o [Helm Repository](https://fluxcd.io/docs/components/source/helmrepositories/) e o [Bucket](https://fluxcd.io/docs/components/source/buckets/).

Para adicionar um GitRepository, é possível utilizar o flux cli:

```
flux create source git <nome> \
  --url=<url-do-repositório-a-ser-sincronizado> \
  --branch=master \
  --interval=10s \
  --export > ./meu-cluster/<nome>-source.yaml
```

Após isso, um manifesto será criado dentro de **./meu-cluster**. Vamos commitar e fazer push para nosso repo:

```
git add -A && git commit -m "feat: add first GitRepository"
git push
```

## 6 Criar um manifesto Kustomization

O [Kustomization Controller](https://fluxcd.io/docs/components/kustomize/kustomization/) é o CRD responsável, resumidamente, por fazer a validação e aplicação dos manifestos K8s através de sua pipeline:

```
flux create kustomization <nome> \
  --target-namespace=default \
  --source=<nome-do-source> \
  --path="./" \
  --prune=true \
  --interval=5m \
  --export > ./meu-cluster/<nome>-kustomization.yaml
```

Ele funciona coletando os workloads que ja estão no cluster e comparando-os com o definido no Source. Se o Source estiver diferente, ele irá fazer o processo de [Reconciliation](https://fluxcd.io/docs/concepts/#reconciliation). Isto é, irá atualizar os workloads no cluster para mante-los igual ao definido no repositório.

A opção **interval** definida agora é o tempo máximo do período de reconciliação, e não deve ser confundido com o intervalo de sincronização do repositório que foi definido no objeto de source anterior.

Também é possível escolher qual o path que será aplicado, caso seu repositório conte com diretórios de dev e prod, é possível especificar, como por exemplo:  **./dev** 


```
git add -A && git commit -m "feat: add first Kustomization"
git push
```

E após isso, é só fazer o push para o repositório e... pronto! O FluxCD já está funcionando no seu cluster como operador GitOps! 🎉🎉🎉

---
*OBS: Esse documento foi totalmente inspirado no [GetStarted](https://fluxcd.io/docs/get-started/) que o próprio FluxCD proporciona.*
