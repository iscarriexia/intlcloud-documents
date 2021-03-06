## 最下層の実装
Tencent Cloud負荷均衡（Cloud Load Balancer、CLB）は、現在、4層と7層におけるCLBサービスを提供しており、Tencentが始めたTencent gateway（TGW）という統一ゲートウェイ製品が提供するCLB能力に基づき、信頼性が高く、拡張性に優れ、性能が高く、攻撃防止能力が高い等の特徴があり、大規模な同時アクセスを実現し、且つ悪意のある攻撃のトラフィックを防止することができます。CLBの製品機能は、CLB提供能力、パブリックネットワークIPの収束、DDoS攻撃防止、QoS、FTPやSIPサポートなどの能力を含みます。
CLBはクラスタ配置を採用しているため、セッションの同期が可能であり、それによりサーバーの単一障害点を解消し、冗長化を向上させ、サービスの安定を保障し、同一地域に複数のデータセンタを配置することでローカル災害復旧を実現することができます。
これからはプラットフォーム、リソース、サービスの3つの次元を通じて、CLBがクラウドプラットフォームのテナント分離、アーキテクチャ災害復旧、リソース災害復旧、攻撃防止などの機能を実現するための重要な技術原理を紹介します。
>?
>- RSとは、CLB装置をバインディングするCVMを指します。
>- VIPとは、CLB装置が外部にサービスを行うIPアドレスを指します。

### テナントの分離
クラウドプラットフォームの重要な能力は、テナントの分離です。クラウドにアクセスするユーザーはいずれも自身のサービスが他の人から影響を受けず、少なくともネットワーク上で分離を実現し、マシンが自由にアクセスされないよう確保されることを望んでいます。一般的な方法はハードウェアにおいて分離することであり、特定のアクセススイッチとvxlanプロトコルを使用することにより分離の目的に達することができますが、この方法には以下の欠点が存在します。
- 特殊なスイッチを使用する必要があります。
- 追加の設備によりvxlanネットワークと通常ネットワークを繋げる必要があり、単一障害点の問題が存在します。
- 既存のネットワーク環境との互換が難しいです。

以上の原因から、CLBではソフトウェアによるソリューションを採用し、IPトンネル + VPCによりテナントの分離を実現しました。
![](//mccdn.qcloud.com/static/img/cf8f46731a218bf7fef43843eef0d4e4/image.png)
上図の左側から、CLBとRSの間はインタラクティブにIPトンネル方式を採用することにより、CVM（RS）によって割り当てられた実際のプライベートネットワークIPと物理ネットワークがお互いに接続されていることが分かります。この方法の実現は簡単で、以前の物理マシンによるソリューションと互換することができますが、以下の欠点が存在します。
- テナント間の分離を実現するために、追加のモジュールが必要です。
- テナント間のプライベートネットワークIPは再利用できず、自由にネットワーキングすることができません。
- IPはプライベートネットワーク上で一意であるため、移行時にRSのIPを変更する必要があり、従ってホットマイグレーションは実現できません。

以上の原因から、Tencent CloudはVPCを開発しました。上図の右側から、同一テナントには1つのVPCIDが割り当てられていることが分かります。VPCでは、クライアントは自由にネットワーキングでき、テナントのネットワーク本体は分離され、具体的な処理はvpc.koカーネルモジュールにより実現することができます。

### IPの収束
CLBについて取り上げるには、LVS（Linux Virtual Server）技術について言及しなければなりません。LVSは全部で以下のモードがあります。
- DRモード：主な欠点はLVSとRSが同一のvlanになければならないことを要求することにあり、配置の制限が大きく、拡張性が劣ります。
- NATモード：主な欠点はRSの戻りパケットがデフォルトのルーティングを使用しなければならないことにあり、拡張性の問題が存在します。従って、当社で最初のLVSクラスタの構築に採用しているのがTUNNELモードです。
- TUNNELモード：各RSにパブリックネットワークIPを割り当てる必要があります。Tencent Cloud及びRSが多いサービスにとって、パブリックネットワークIPは大きな課題です。

以上の原因から、Tencent CloudはCLBを研究開発しました。下図はLVSソリューションとCLBソリューションのインスタンスの概略図で、両者の主な違いは以下のとおりです。
- CLBはRSにパブリックネットワークIPを割り当てる必要はなく、IPを収束する役割をします。
- アウトバウンドトラフィックは依然としてCLBを通過するため、問題の確定がより簡単で、サービストラフィックに対して閉ループが形成され、足がかりの役割をします。
![](//mccdn.qcloud.com/static/img/e4575f5f414666505d8c1a7cdea2c6f0/image.png)

### 高い信頼性の実装
高い信頼性はクラウドサービスを測る重要なメトリックです。CLBは以下のソリューションを実現することにより、お客様に信頼性の高いサービスを提供します。
- [クラスタ災害復旧](#jqrz)
- [session同期](#jqrz)
- [リソース分離](#zygl)
- [DDoS攻撃防止](#kgj)

<span id="jqrz"></span>
#### クラスタ災害復旧とsession同期
クラスタ災害復旧とは、クラスタ中の1台のサーバーがダウンしてもクラスタ全体のサービス能力に影響を与えないことを指します。従来のクラスタ災害復旧に採用していたのはHAモード（vrrpプロトコル）ですが、一般的なオープンソースソリューション（例：LVS）の欠点は以下のとおりです。
- 1つのクラスタでは半分のサーバーしか同時に動作することができず、残りの半分のマシンはコールドスタンバイ状態になっています。
- サーバーダウン後の切り替え速度が、相対的に遅いです。

CLBはospf動的ルーティングプロトコルを採用することによりクラスタ災害復旧を実現し、1台のマシンに故障が発生した場合、ospfプロトコルは10 s以内にマシンをクラスタから除去することを保証することができます。CLBの1つのクラスタが2つのアクセススイッチに置かれ、且つラック間の災害復旧を保証するため、片方のアクセススイッチに故障が発生または片方のラックが停電しても、本クラスタのサービスは影響を受けません。
クラスタ災害復旧はCLBクラスタの可用性を確保しますが、お客様にとっては、サーバーがダウンした場合、このマシンが除去されたとしても新しい接続がこのマシンにならないよう保証することができるだけで、長時間の接続は切断されます。この問題を解決するため、当社ではクラスタ内のsession接続の定期的な同期を実現し、他のサーバーが故障したマシンのパケットを引き継んだ時に正確にsessionを見つけることができ、正常にサービスを提供することを保証します。
![](//mccdn.qcloud.com/static/img/4cdd6084a39561e04539a8866374bb24/image.png)
要約すると、
- LVS：接続状態に変化があった場合に同期します。長い接続と短い接続が共存する状況において、短い接続のサービスの同期はトラフィックが非常に大きいため、正常な転送に影響を与えます。
- CLB：各接続を確立してから5秒後に同期し、接続が5秒以内の場合、同期しません。長い接続にのみ同期します。
![](//mccdn.qcloud.com/static/img/397479668381a345c8bae877e4aa4ff3/image.png)

<span id="zygl"></span>
#### リソース分離
リソース分離は別のサービスが攻撃を受け、CLBに高負荷が発生した時に、他のサービスが影響を受けないよう保証することができます。具体的な実現方法は以下のとおりです。
- 定期的に（5s）CLBが構成された高負荷しきい値に達しているかどうかを検出し、しきい値に達している場合、リソースの分離が有効になります。
- CLBは各サービスのトラフィック、パケット量、接続数を検出し、上限を超えた場合は破棄し、CLBサーバーが実際に高負荷に達して他のサービスに影響を及ぼさないよう保証します。
- 正常な状況において、リソースの分離機能はオフになっています。サービスがトラフィックを急増させた場合、または特定のサービスが攻撃を受けたことによりCLBがしきい値に達した場合にのみ5秒以内にこの機能がオンになり、正常なサービス転送が影響を受けないよう保証します。
- リソース分離は一般的なトークンバケットアルゴリズムを使用します。その動作過程は、定期的にバケットに一定量のトークンを置き、パケットが受信されるたびに1つのトークンが消費され、一定期間でトークンが使い果たされるとパケットロスが発生します。
![](//mccdn.qcloud.com/static/img/86cd36ef04f3200b8d0c591b0c4e7675/image.png)

<span id="kgj"></span>
#### DDoS攻撃防止
クラスタ災害復旧とリソース分離はいずれもCLBプラットフォームを保護することですが、単一のサービスについては、攻撃を受けた場合に被害を受ける確率は100%であり、CLBではこのような状況が発生することが許容されません。DDos攻撃について、Tencent Cloudでは専門の[DDos防御](https://cloud.tencent.com/product/ddos)によりサービスを保護していますが、その検出時間は10 sであり、DDoS防御の機能が有効になる前にお客様のRSは破壊されてしまう恐れがあります。この10 s以内の問題を解決するため、当社ではsynproxy機能を開発しました。具体的な実現方法は、以下のとおりです。
- CLBがクライアントの3ウェイハンドシェイクリクエストを受信した時、3ウェイハンドシェイクをプロキシし、データパケットが到着するまでRSは影響されません。
- 1つ目のパケットが到着した時、CLBはそれをキャッシュしてから、RSと3ウェイハンドシェイクを行います。
- ハンドシェイクが完了した後、キャッシュしたデータパケットをRSに送信し、その後のプロセスはデータパケットをトランスペアレント伝送します。

この方法ではDDoS攻撃がRSに到達せず、CLBが負荷を受けることを保証します。CLB本体の負荷能力は強く、また、クラスタモードであり、同時に資源の分離能力を有しているため、一般的な状況において10 s以内にCLBマシンを破壊することは難しいです。
![](//mccdn.qcloud.com/static/img/5c96f1c2548dd15bd00d0ff01b63eddf/image.png)

## Tencent Cloud CLBの特徴
Tencent Cloud CLBは、現在、4層と7層のプロトコルの転送能力を提供しています。
- 4層CLBはリスナーのTCPとUDPに対応しています。
- 7層CLBはリスナーのHTTP/HTTPS能力に対応しています。

### 4層CLB
4層CLBはCLBが最も初期に実現したソリューションであり、CLB製品に必須の機能です。基本的な原理はCLBでポートにより様々なサービスを区別することであり、転送規則（rule）のkeyはvip:vport:protocolです。現在、Tencent Cloudが最も多く使用しているのはこのCLB方式ですが、Tencent CloudにおいてVIPは同じ開発業者に属しており、異なる開発業者間のトラフィックは厳密に分離されています。
![](//mccdn.qcloud.com/static/img/bb969f908e3931c61267c316e6e4f909/image.png)

### 7層CLB
非デイリーレンタル型のソリューションは通常の7層CLBサービスに対応することができますが、sessionとcookieのニーズを有する7層ユーザーにとっては自身でnginxを確立してリバースプロキシを行わなければならず、無駄であるだけでなく、信頼性も影響を受けます。
![](//mccdn.qcloud.com/static/img/6d385cd8c23ca392c36540eff8689e5c/image.png)
以上の状況から、7層CLBは2つの代替実現案があります。
- ソリューション1：パブリックネットワークIPを直接nginxマシン上に作成し、nginxクラスタを構築する。
- ソリューション2：nginxクラスタを4層CLBの下で接続する。

ソリューション1の弱点は、DDoS攻撃に対して打つ手がないことです。Tencent Cloudの場合、1つのVIPが同時に4層と7層にアクセスできる必要があるため、CLBはソリューション2を採用しました。また、ソリューション2は動的容量拡張と圧縮が便利で、サービスが急速に容量拡張するシナリオに対応することができます。
しかし、nginx本体はリバースプロキシによりCLVを実現するため、Tencent Cloudに存在する問題は、VPCネットワークは仮想ネットワークであり、物理ネットワークとの間はホストを介して繋がっているため、7層CLBとRSの間でnginxのリバースプロキシ機能を直接使用することはできないことです。従って、当社では7層CLBにgreトンネルとIPトンネルとRS間のインタラクティブの実装を行うためのl7.koカーネルモジュールを挿入ししました。
![](//mccdn.qcloud.com/static/img/9874ed32509218619ef4cea119bc3790/image.png)

