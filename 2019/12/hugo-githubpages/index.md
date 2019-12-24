# Criando Github Pages com Hugo


### Configurando um site estático com hugo e github pages

Estou reunindo as configurações que eu usei para montar este site, mas sempre é bom consultar a [doc oficial](https://gohugo.io/documentation/).

### 1. Instalando o Hugo. 
Foi usado o binário do pacote para linux, a versão usada foi a v0.62.0
```
$ https://github.com/gohugoio/hugo/releases/download/v0.62.0/hugo_0.62.0_Linux-64bit.tar.gz
$ tar zxvf hugo_0.62.0_Linux-64bit.tar.gz && sudo mv hugo /usr/local/bin/
$ hugo version
```

### 2. Criar o site
```
$ hugo new site escolha_um_nome
$ cd nome_escolhido && git init .
```

### 3. Adicionando tema

[temas oficias](https://themes.gohugo.io/)

```
$ cd themes
$ git clone https://github.com/dillonzq/LoveIt.git
```

Copiar o exemplo de conf para a raiz do site
```
$ cp -r LoveIt/exampleSite/* ../
```

### 4. Inciar o Hugo

Agora basta executar o comando abaixo e depois edite o arquivo “config.toml” com suas preferencias 
```
$ hugo server
```

### 5. Criando um post
```
hugo new post/hugo-githubpages.md
```

### 6. Salvando as mundanças

 Para salvar as modificacoes feitas no hugo vamos criar um repositorio privado no github e fazer um push.
```
 $ echo "# confs do hugo" >> README.md
 $ git add -A
 $ git commit -m "Adicionar arquivos basicos do Hugo"
 $ git remote add origin https://github.com/usuario/repositorio.git
 $ git push -u origin master
 ```
### 7. Criando github pages

Agora devemos criar um repositorio publico com o seguinte nome: "usuario.github.io"
```
$ git submodule add -b master https://github.com/usuario/usuario.github.io.git public
$ hugo -D
$ cd public/
$ git add .
$ git commit -m "Subindo o blog"
$ git push origin master
```