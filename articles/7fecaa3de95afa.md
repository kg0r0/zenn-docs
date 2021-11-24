---
title: "Chrome DevToolsのRecoderためしてみた"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chrome", "javascript"]
published: true
---
# はじめに
Chrome DevToolsのRecoderをためしてみただけの記事です。
詳細は以下の公式ページを参照ください。
https://developer.chrome.com/docs/devtools/recorder/
なお、今回は使用したChromeのバージョンは``Version 98.0.4724.0 (Official Build) canary (x86_64)``です。

# Recording
まずは任意のWebページで開発者ツールを開き、``More options > More tools > Recorder``の順でRecoderを開きます。
![](/images/7fecaa3de95afa/recorder.png)
つづいて``＋``ボタンを押下し、``[RECORDING NAME]``を入力して``[Start a new recording]``を押下します。
![](/images/7fecaa3de95afa/start_recording.png)
ブラウザから画面を操作すると次々に操作がRecorderパネルに記録されていきます。
一通り操作が終わったら、``[End recording]``を押下します。
![](/images/7fecaa3de95afa/end_recording.png)
最後に、プルダウンから作成したRecordingを選択して``[Replay]``を押下することで、記録した操作がブラウザで実行されます。
![](/images/7fecaa3de95afa/rename.png)

# Puppeteer
RecordingはPuppeteer Scriptとしてエクスポートすることができます。これにより記録した操作を他者と共有することが可能になります。
エクスポートしたいRecordingを選択した状態で``[Export as Pupeteer script]``を押下します。
![](/images/7fecaa3de95afa/export.png)
エクスポートしたJavaScriptファイルは以下のようにpuppeteerをインストールしたのちに実行します。
```bash
$ npm i puppeteer
$ node [PUPPETEER SCRIPT]
```
うまく実行できるとブラウザが起動して操作を再生することができます。
Puppeteerの[Debugging tips](https://github.com/puppeteer/puppeteer#debugging-tips)を参考に、エクスポートしたスクリプトに``headles: false``オプションを追加すると以下のように記録した操作が再生されていることがわかりやすく確認できます。
![](/images/7fecaa3de95afa/script.png)

# おわりに
Chrome DevToolsのRecoderは色々便利な使い道がありそうなので今後も探っていきたいと思います。
今回利用したスクリプトは以下のGitHubリポジトリに編集前後の両方置いているので良ければ参照ください。
https://github.com/kg0r0/coffee-checkout

# 参考
- https://developer.chrome.com/docs/devtools/recorder/
- https://github.com/puppeteer/puppeteer
- https://github.com/kg0r0/coffee-checkout