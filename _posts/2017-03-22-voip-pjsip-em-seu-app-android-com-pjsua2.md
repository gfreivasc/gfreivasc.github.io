---
layout: post
title: VoIP PJSIP em seu app Android com PJSUA2
date: 2017-03-22 19:00:00+0300
categories: android libraries pt-BR
related_image: "https://cdn-images-1.medium.com/max/2000/1*WFGZgkF1HuA8vbzcpADNqA.jpeg"
---
Compilando e incluindo a API de VoIP PJSUA2 no seu projeto

<figure class="align-center">
  <img src="{{ page.related_image }}" alt="">
</figure>

Não vou falar sobre como criar um aplicativo VoIP, mas sobre como incluir as funcionalidades do PJSUA2 no seu projeto Android. Não é uma tarefa simples como na maioria das vezes que você quer incluir um framework novo, visto que você tem que compilar *manualmente* a biblioteca para cada arquitetura que você pretende dar suporte.

Vou complementar o guia oficial do [pjsip](https://trac.pjsip.org/repos/wiki/Getting-Started/Android) com informações atualizadas e detalhes não muito explícitos, que podem dar dor de cabeça para quem esteja pensando em trabalhar com a API.

## Requisitos

Este texto foi feito para compilar o PJSUA2 em sistemas UNIX.

É interessante que você esteja trabalhando com o [Android Studio](https://developer.android.com/studio/index.html?hl=pt-br) a esse ponto do campeonato. Não vejo motivo para não estar. Vou considerar que esteja daqui para frente, por que é mais simples (para você).

É necessário que você tenha baixado a NDK do Android na sua máquina, no mínimo na versão r8b. Você pode baixar pelo próprio Android Studio:

1. Vá em ***File > Settings***.

2. ***Appearance & Behavior > System Settings > Android SDK***

3. ***SDK Tools***

4. Selecione a opção ***NDK***.

5. Confirme em ***Apply*** ou ***OK***.

O NDK será salvo na pasta ndk_bundle na mesmo local onde são salvas as SDKs. (No meu caso, ~/Android/Sdk/ndk_bundle).

Além disso, tenha o [SWIG](http://www.swig.org/download.html) 2.0.5 ou mais recente.

Por último, obviamente, o [código fonte do PJSIP](https://trac.pjsip.org/repos/wiki/Getting-Started/Download-Source#GettingfromSubversiontrunk).

## Quais arquiteturas e versão usar?

Será necessário especificar quais arquiteturas deverão ser utilizadas. A arquitetura armeabi era padrão até o Android KitKat (API 19), Posteriormente têm-se utilizado armeabi-v7a . Além dessas opções, há outras para arquiteturas MIPS e Intel, como o famoso x86 . Você pode optar por dar suporte a múltiplas arquiteturas.

Além disso é importante definir qual a versão mínima da API do Android a qual o app dará suporte. Você compilará o PJSUA2 pra essa versão, e a biblioteca será compatível com versões posteriores. É necessário que você escolha uma das opções disponíveis na pasta ndk_bundle/platforms que foi baixada junto com o NDK.

Na minha versão local, as opções são da API 9 à 24, exceto as versões 10, 11 e 20. Como eu queria que funcionasse a partir da API 19, defini compilar a biblioteca com target sendo a API 19.

## Preparando a build

Verifique se a header android_alarms.h está presente na pasta include/linux/ da versão e arquiteturas escolhidas. Precisamente em …/ndk_bundle/platforms/android-*/arch-*/. Por exemplo, pra API 19:

* Para armeabi, armeabi-v7a, etc — android-19/arch-arm/include/linux

* Para x86 e x86_64 — android-19/arch-x86/include/linux

* Para MIPS — android-19/arch-mips/include/linux

Se você não encontrar, [baixe a original](https://android.googlesource.com/platform/external/kernel-headers/+/donut-release/original/linux/android_alarm.h) e cole nas pastas das arquiteturas escolhidas.

Defina a variável de ambiente ANDROID_NDK_ROOT para a pasta da NDK. Se baixou pelo Android Studio, esta é a pasta ndk_bundle já mencionada.

    $ export ANDROID_NDK_ROOT=…/ndk_bundle

Supondo que você já tenha baixado [o código fonte do PJSIP](https://trac.pjsip.org/repos/wiki/Getting-Started/Download-Source#GettingfromSubversiontrunk), da pasta raiz, vá até pjlib/include/pj e crie o arquivo config_site.h com o seguinte:

```cpp
/* Activate Android specific settings in the ‘config_site_sample.h’ */
#define PJ_CONFIG_ANDROID 1
#include <pj/config_site_sample.h>
```

## Compilando o código fonte

Com tudo pronto, vamos ao passo-a-passo. Será feito um processo de compilação para cada arquitetura dentre as escolhidas. Se armeabi for umas das escolhidas, deixe ela por último.

Primeiro, volte para o diretório raiz. defina a configuração para a arquitetura e API (Por exemplo, x86 com a api 19):

    $ TARGET_ABI=x86 APP_PLATFORM=android-19 ./configure-android -use-ndk-cflags

Caso apareça o erro ***configure-android error: compiler not found, please check environment settings (TARGET_ABI, etc)***, rode o comando com mais uma flag:

    $ NDK_TOOLCHAIN_VERSION=4.9 TARGET_ABI=x86 APP_PLATFORM=android-19 ./configure-android --use-ndk-cflags

Após o termino da configuração, faça a primeira compilação com o *make*:

    $ make dep & make clean & make

Espere até terminar e depois vá na pasta pjsip-apps/src/swig para completar o processo.

    $ make
    ...
    $ make clean

Isso irá gerar tanto a .so do pjsua2 quanto as interfaces Java que irão se comunicar com ela. Porém a .so ficará na pasta destinada para a arquitetura armeabi ao invés da pasta para x86 , sendo necessário mover para a pasta correta. Ainda da pasta do swig rode:

    $ mv java/android/app/src/main/jniLibs/armeabi java/android/app/src/main/jniLibs/x86

Repita todo o processo desde a configuração para cada arquitetura, deixando a armeabi por último. Quando compilar para armeabi , não será necessário mover nada.

## Usando os arquivos recém gerados no seu projeto

Terminamos a compilação, mas agora precisamos saber como utilizar os arquivos compilados. Eles foram criados no exemplo que vem com o código, na pasta `<raiz-pjsip>/pjsip-apps/src/swig/java/android/`

Copie todo o código do projeto no pacote org.pjsip.pjsua2 , exceto a pasta app , que contém o exemplo que não vamos precisar.

    $ cd MyApp/app/src/main/java
    $ cp -r …/pjsip-apps/src/swig/java/android/app/src/main/java/org/ .
    # Removendo a pasta desnecessária
    $ rm -r org/pjsip/pjsua2/app

Copie, por último, a pasta jniLibs que contém os .so de cada arquitetura, da seguinte forma, por exemplo:

    jniLibs
     \ armeabi
     | \ libpjsua2.so
     \ armeabi-v7a
     | \ libpjsua2.so
     \ x86
       \ libpjsua2.so

Se você moveu as pastas corretamente na parte das compilações.

    $ cd MyApp/app/serc/main
    $ cp -r …/pjsip-apps/src/swig/java/android/app/src/main/jniLibs .

E os arquivos estarão corretamente colocados no seu projeto.

Porém é necessário inicializar a lib para que ela funcione. Na sua classe Application, carregue antes de qualquer procedimento da PJSUA2:

```java
package com.me.myapp;

import android.app.Application;

public class MyApp extends Application {
    static {
        System.loadLibrary("pjsua2");
    }

    // ...
}
```

Agora deve estar tudo pronto para rodar sua aplicação. Daqui pra frente, dê uma lida na [documentação oficial](http://www.pjsip.org/docs/book-latest/html/index.html) para saber como agir. Até a próxima!
