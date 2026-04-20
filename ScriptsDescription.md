# スクリプト詳細説明

このドキュメントでは、プロジェクト内の各C#スクリプトの処理内容と役割について詳細に説明します。

## 1. DTXFileLoadTest.cs
- **役割**: DTXファイルの読み込みテストを行うためのスクリプト。
- **処理内容**: 起動時に `DTXInputOutput` コンポーネントを取得し、特定のDTXファイル（例: `"Sing Alive (Full Version)/mstr.dtx"`）の読み込みを開始します。`Update` メソッド内でファイルの読み込み完了状態 (`IsSongReady()`) を監視します。

## 2. DTXHelper.cs
- **役割**: DTXファイルの解析や音声ファイルの取得などで使用されるユーティリティクラス。
- **処理内容**:
  - **Base36ToInt**: 36進数の文字列を整数に変換する機能（DTXのチップ番号解析などに使用）。
  - **GetAudioClip**: `UnityWebRequestMultimedia` を用いて、指定されたパスから音声ファイル（OGG形式など）を非同期で取得するコルーチン。
  - **ReadInputFile / ProcessInputBuffer**: テキストファイルを読み込み、改行コードの統一やタブのスペースへの置換などの前処理を行います。

## 3. DTXInputOutput.cs
- **役割**: DTXファイルのパース（構文解析）と楽曲データの構築を担当するコアスクリプト。
- **処理内容**:
  - `#TITLE`、`#BPM`、`#WAV` などのDTXコマンドを解析し、`CommandObject` や `MusicInfo` といった構造体に格納します。
  - 楽曲のチップ（ノーツ）配置データを読み取り、レーンやタイミング（拍・小節）、BPMの変化を計算してリスト化します。
  - 楽曲データの準備が完了したかどうかを判定する機能（`IsSongReady`）を提供します。

## 4. InputDevice.cs (Multimedia.Midi名前空間)
- **役割**: WindowsのマルチメディアAPI (`winmm.dll`) を利用して、MIDI入力デバイスとの通信を管理するスクリプト。
- **処理内容**:
  - MIDIデバイスのオープン、クローズ、録音（入力の待機）の開始と停止を行います。
  - MIDIデバイスから送られてくるシステムエクスクルーシブメッセージやショートメッセージを受信・解析し、対応するイベント（`ChannelMessageReceived` など）を発火させます。

## 5. Messages/*.cs (Multimedia.Midi名前空間)
- **役割**: MIDIメッセージの各種フォーマットを表現するためのクラス群。
- **処理内容**:
  - **IMidiMessage / IMidiMessageVisitor**: MIDIメッセージのインターフェースと、Visitorパターン用のインターフェース。
  - **ChannelMessage.cs**: チャンネルメッセージ（ノートオン/オフ、コントロールチェンジなど）のデータ構造とパース処理。
  - **SysExMessage.cs**: システムエクスクルーシブメッセージのデータ構造。
  - **SysRealtimeMessage.cs**, **SysCommonMessage.cs**, **ShortMessage.cs**, **MetaMessage.cs**: その他のMIDIメッセージの各種定義と処理。

## 6. MIDIControllerTest.cs
- **役割**: MIDIコントローラー（電子ドラムやMIDIキーボードなど）の入力テストを行うスクリプト。
- **処理内容**:
  - 起動時に接続されているMIDI入力デバイス数を取得し、最初のデバイス（インデックス0）を初期化してリスニングを開始します。
  - チャンネルメッセージを受信した際に `OnChannelMessage` メソッドが呼ばれ、押されたノート（Note）やベロシティ（Velocity）の情報をログに出力します。

## 7. SongList.cs
- **役割**: 楽曲選択画面のUIリストを構築・管理するスクリプト。
- **処理内容**:
  - `SongManager` が楽曲リストの読み込みを完了するのを待機します。
  - 読み込みが完了すると、`SongManager` が保持する楽曲情報 (`SongInfo`) のリストを基に、プレハブ (`songButtonPrefab`) を複製してUI上にボタンを生成します。
  - ボタンがクリックされた際に、`StageManager` を通じてその楽曲の再生処理を呼び出すようリスナーを登録します。

## 8. SongManager.cs
- **役割**: ディレクトリ内を走査して、利用可能な楽曲リストを読み込み・管理するスクリプト。
- **処理内容**:
  - 指定されたフォルダ（`StreamingAssets` 以下の `songFolderPath`）内のファイルを再帰的に検索します。
  - `SET.def`（複数難易度をまとめるファイル）や `*.dtx` ファイルを見つけると、`DTXHelper` を使ってファイルの中身を読み込み、楽曲名、アーティスト名、難易度情報などを解析します。
  - 解析した情報を `SongInfo` オブジェクトとしてリストに格納し、他のスクリプトから参照できるようにします。

## 9. SoundManager.cs
- **役割**: ゲーム内のサウンド（BGMやSE）再生を効率的に行うためのオブジェクトプーリングシステム。
- **処理内容**:
  - 予め複数の `AudioSource` をプールとして生成しておき（`audioSourcePool`）、音を再生するリクエストがあった際にプールから空いている `AudioSource` を取得して再生します。
  - `Update` メソッドで各 `AudioSource` の再生終了時刻（`scheduledTime`）をチェックし、再生が終わったものはリセットしてプールに返却します。
  - `PlayScheduled` を使用し、指定されたDSPタイムで正確に音声を再生する仕組みを提供しています。

## 10. StageManager.cs
- **役割**: 選択された楽曲のロードとゲームプレイ（ステージ）の進行を管理するスクリプト。
- **処理内容**:
  - UI等から `PlaySong` メソッドが呼ばれると、指定された難易度のDTXファイルのパスを構築します。
  - `DTXInputOutput` コンポーネントにファイルのロードを指示し、ロードが完了するまでコルーチン (`LoadAndPlaySelectedSong`) で待機します。
  - 準備が完了した後、自動再生モード (`autoPlay`) などのフラグに応じて楽曲の再生処理を開始させます。