---
layout: post
title: "How to make the Telegram bot"
date: 2016-05-06 00:00:00 +0900
comments: true
category: tutorial
---

이 글은 AWS Lambda + Gateway에서 ClojureScript + Node.js를 사용해 텔레그램 봇을 만드는 과정을 설명한 글 입니다[1].

AWS를 사용한 적이 없다면 -1년간 [공짜](https://aws.amazon.com/free/)니깐 일단- 가입하고,  
[여기](https://obviate.io/2015/08/05/tutorial-aws-api-gateway-to-lambda-to-dynamodb/)에서
알려준대로 따라하면 AWS Lambda + Gateway 셋업이 끝납니다[2].

위 튜토리얼에서 사용한 [자바스크립트 코드](https://gist.github.com/ShakataGaNai/6027b4c684c294f3fcef)가
동작하는 것을 확인 했다면,  
이제 이 코드를 CLJS로 옮겨 적고 AWS Lambda에서 잘 동작하는지 확인할 차례입니다[3].  
코드가 잘 옮겨 졌으면, 마지막으로 앞서 옮겨 적은 코드를 참고해
[텔레그램 봇: 자바스크립트 코드](https://github.com/ShakataGaNai/poc-telegram-bot-aws-lambda/blob/master/telegramEcho.js)를 CLJS로 옮겨 적습니다.

[1] 대부분 내용은 '[PoC Telegram Bot running in AWS Lambda](https://snowulf.com/2015/08/28/tutorial-poc-telegram-bot-running-in-aws-lambda/)'을 참조 했습니다.  
[2] DB를 사용하지 않으니 안만들어도 됩니다.  
[3] 이미 AWS 람다를 위한 [라이브러리](https://github.com/uswitch/lambada)가 있지만, 차근차근 해보고 싶어서 사용하지 않았습니다.

## CLJS

먼저, 아래와 같이 템플릿으로 부터 CLJS 프로젝트를 생성 합니다.

``` sh
$ lein new mies pow
```

## CLJS + Node.js

다음으로 Node.js에 맞게 프로젝트를 수정합니다.

./src/pow/core.cljs: 

``` clojure
(ns pow.core
  (:require [cljs.nodejs :as nodejs]))

(nodejs/enable-util-print!)

(defn -main [& args]
  (println "Hello world!"))

(set! *main-cli-fn* -main)
```

./scripts/nodejs:

``` sh
#!/bin/sh
rlwrap lein trampoline run -m clojure.main scripts/nodejs.clj
```

./scripts/nodejs.clj:

``` clojure
(require '[cljs.build.api :as b])

(println "Building ...")

(let [start (System/nanoTime)]
  (b/build "src"
           {:main       'pow.core
            :output-to  "out/main.js"
            :output-dir "out/"
            :target     :nodejs})
  (println "... done. Elapsed" (/ (- (System/nanoTime) start) 1e9) "seconds"))
```

위와 같이 수정하면 아래와 같이 빌드하고 실행할 수 있습니다.

``` sh
$ cd pow
$ scripts/nodejs
$ node out/main.js
> Hello world!
```

## CLJS + Node.js + AWS Lambda

다음으로 프로젝트를 AWS Lambda 맞도록 수정해야 합니다.

[튜토리얼](https://obviate.io/2015/08/05/tutorial-aws-api-gateway-to-lambda-to-dynamodb/)에서
AWS Lambda 생성할 때 Handler란에 "index.handle"라고 적었기 때문에  
index.js 파일의 export.handle에 우리의 헨들러 함수를 반환해야 합니다.  
다행히 CLJS의 함수는 자바스크립트의 함수와 같기 때문에 CLJS 함수를 그대로 연결해도 됩니다.

./src/pow/core.cljs:

``` clojure
(ns pow.core
  (:require [cljs.nodejs :as nodejs]))

(nodejs/enable-util-print!)

(defn -main [event context]
  (println (. js/JSON (stringify event)))
  (println (. js/JSON (stringify context))))

;; to fake CLJS compiler
(defn none [& _] nil)
(set! *main-cli-fn* none)
```

./scripts/aws-lambda:

``` sh
#!/bin/sh
rlwrap lein trampoline run -m clojure.main scripts/aws-lambda.clj
echo "exports.handler = pow.core._main;" >> index.js
rm -rf $(basename $(PWD)).zip
zip -r $(basename $(PWD)) index.js out/ node_modules/ > /dev/null
```

코드가 많이 조잡합니다.
귀찮아서...

./scripts/aws-lambda.clj:

``` clojure
(require '[cljs.build.api :as b])

(println "Building ...")

(let [start (System/nanoTime)]
  (b/build "src"
           {:main       'pow.core
            :output-to  "index.js"
            :output-dir "out/"
            :target     :nodejs})
  (println "... done. Elapsed" (/ (- (System/nanoTime) start) 1e9) "seconds"))
```

``` sh
$ scripts/aws-lambda
```

생성된 zip 파일을 업로드 하고,
[튜토리얼](https://obviate.io/2015/08/05/tutorial-aws-api-gateway-to-lambda-to-dynamodb/)에서
테스트 했던 것 처럼 테스트해 로그가 잘 찍히는지 확인합니다.

## CLJS + Node.js + AWS Lambda + Telegram Bot

끝으로 다음과 같이 봇 코드를 옮겨 적습니다.

./src/pow/core.cljs:

``` clojure
(ns pow.core
  (:require [cljs.nodejs :as nodejs]))

(nodejs/enable-util-print!)

(defonce https       (nodejs/require "https"))
(defonce querystring (nodejs/require "querystring"))

(defn -main [event context]
  (println "Request received:" (js->clj event :keywordize-keys true))
  (let [api-key "YOUR-BOT-API-KEY"
        text    (str "Hello " (.. event -message -from -first_name) ", "
                     "I'm ClojureScript version Bot. "
                     "You said \"" (.. event -message -text) "\".")
        data    (->> {:chat_id             (.. event -message -from -id)
                      :reply_to_message_id (.. event -message -message_id)
                      :text                text}
                     (clj->js)
                     (.stringify querystring))
        header  (clj->js {:hostname "api.telegram.org"
                          :port     443
                          :path     (str "/bot" api-key "/sendMessage")
                          :method   "POST"
                          :headers  (clj->js {:Content-Type   "application/x-www-form-urlencoded"
                                              :Content-Length (.-length data)})})
        request (. https (request header (fn [res]
                                           (. res (setEncoding "utf8"))
                                           (. res (on "data" #(println "Response:" %)))
                                           (. res (on "end"  #(. context (succeed)))))))]
    (. request (write data))
    (. request (end)))
  (println "Done."))

;; to fake CLJS compiler
(defn none [& _] nil)
(set! *main-cli-fn* none)
```
