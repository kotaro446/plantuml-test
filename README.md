# EC関連問合せMockアプリケーション

## シーケンス図

```plantuml
@startuml
!theme plain
skinparam backgroundColor white
skinparam sequence {
    ArrowColor #666666
    LifeLineBorderColor #cccccc
    LifeLineBackgroundColor #f8f9fa
    ParticipantBorderColor #cccccc
    ParticipantBackgroundColor #ffffff
    ActorBorderColor #cccccc
    ActorBackgroundColor #ffffff
    BoxBorderColor #cccccc
    BoxBackgroundColor #f8f9fa
}
skinparam note {
    BackgroundColor #fffbf0
    BorderColor #cccccc
}

participant "クライアント\n(店舗レジ・システム)" as Client
participant "main.py\n(Flaskアプリ)" as Main
participant "ec_ticket.py\n(メイン処理)" as EC
participant "ec_ticket_library.py\n(検証・共通処理)" as Validator
participant "ec_ticket_nomal.py\n(通常問合せ処理)" as Normal
participant "ec_ticket_refund.py\n(払戻し処理)" as Refund
participant "ec_ticket_complete.py\n(完了通知処理)" as Complete
participant "ec_ticket_kenmen.py\n(券面情報処理)" as Kenmen
participant "ec_ticket_xmldef.py\n(XML生成)" as XMLDef
participant "logger\n(ログ出力)" as Logger
participant "recorder\n(記録)" as Recorder

note over Client, Recorder
    【EC関連問合せ・発券・払戻し・完了通知 Mockサーバ】
end note

Client -> Main : POST /inkessai/recvrequest.do\nリクエストデータ(form-data)
activate Main
Main -> Recorder : save_request()\nリクエスト記録
Main -> Logger : リクエスト受信ログ
Main -> EC : mock_ecticket_ms_main(request_data)
activate EC

EC -> Validator : ec_ticket_validate_check(request_data)\n入力値検証
activate Validator
note right of Validator
    ・要求区分チェック
    ・必須項目チェック
    ・検索キー/バーコードNoチェック
    ・電文区分・業務区分エラー
end note
Validator --> EC : check_result ("00":正常, その他:エラー)
deactivate Validator

alt 検証エラー
    EC -> Logger : エラーログ出力
    EC --> XMLDef : エラーレスポンス生成
else 検証OK
    EC -> EC : make_ticket_info(request_data)\nチケット情報作成
    activate EC

    EC -> Validator : get_interface_info_dictionary()\n基本辞書作成
    activate Validator
    Validator --> EC : interface_info
    deactivate Validator
    
    EC -> Validator : get_order_info_dictionary()\n注文情報辞書作成
    activate Validator
    Validator --> EC : order_info
    deactivate Validator
    
    EC -> Validator : get_payback_info_dictionary()\n払戻情報辞書作成
    activate Validator
    Validator --> EC : payback_info
    deactivate Validator
    
    EC -> Validator : get_order_details_dictionary()\n注文詳細辞書作成\n★今回修正：要求区分B,H,J対応
    activate Validator
    Validator --> EC : order_details
    deactivate Validator

    EC -> Validator : get_test_option(request_data)\nマッピング番号取得
    activate Validator
    note right of Validator
        検索キー/バーコードから
        マッピング番号を判定
    end note
    Validator --> EC : (mapping_no, mapping_str)
    deactivate Validator

    alt mapping_no == 0 (No.1_EC関連問合せ応答)
        EC -> Normal : get_ec_nomal_field(request_data, mapping_str)
        activate Normal
        Normal -> Normal : split_bits()\nマッピング文字列解析
        Normal -> Normal : get_interface_info()\n処理結果・応答結果設定
        Normal -> Normal : get_order_info()\n支払種別・納品種別等設定
        Normal -> Validator : get_ticket_info()\nチケット情報生成
        activate Validator
        Validator --> Normal : ticket_info
        deactivate Validator
        note right of Normal #ffcccc
            ★今回修正箇所
            要求区分B,H,Jの場合
        end note
        Normal -> Validator : get_order_details_dictionary()
        activate Validator
        Validator --> Normal : order_details ★追加
        deactivate Validator
        Normal --> EC : (interface_info, order_info, ticket_info, order_details, httpstatus)
        deactivate Normal

    else mapping_no == 1 (No.2_EC関連払戻し応答)
        EC -> Refund : get_ec_refund_field(request_data, mapping_str)
        activate Refund
        Refund -> Refund : split_bits()\nマッピング文字列解析
        Refund -> Refund : get_interface_info()\n処理結果設定
        Refund -> Refund : get_payback_info()\n払戻金額・期間設定
        Refund --> EC : (interface_info, payback_info, httpstatus)
        deactivate Refund

    else mapping_no == 2 (No.3_EC関連完了応答)
        EC -> Complete : get_ec_complete_field(request_data, mapping_str)
        activate Complete
        Complete -> Complete : split_bits()\nマッピング文字列解析
        Complete -> Complete : get_interface_info()\n処理結果・支払方法区分設定
        Complete -> Complete : get_order_info()\n応答結果・支払種別等設定
        Complete -> Validator : get_ticket_info()\nチケット情報生成
        activate Validator
        Validator --> Complete : ticket_info
        deactivate Validator
        note right of Complete #ffcccc
            ★今回修正箇所
            要求区分B,H,Jの場合
        end note
        Complete -> Validator : get_order_details_dictionary()
        activate Validator
        Validator --> Complete : order_details ★追加
        deactivate Validator
        Complete --> EC : (interface_info, order_info, ticket_info, order_details, httpstatus)
        deactivate Complete

    else mapping_no == 3 (No.4_EC関連問合せ応答_券面情報)
        EC -> Kenmen : get_ec_kenmen_field(request_data, mapping_str)
        activate Kenmen
        Kenmen -> Kenmen : split_bits()\nマッピング文字列解析
        Kenmen -> Kenmen : get_order_info()\n支払種別・納品種別等設定
        Kenmen -> Kenmen : get_ticket_info()\n券面情報・バーコード生成
        note right of Kenmen #ffcccc
            ★今回修正箇所
            要求区分B,H,Jの場合
        end note
        Kenmen -> Validator : get_order_details_dictionary()
        activate Validator
        Validator --> Kenmen : order_details ★追加
        deactivate Validator
        Kenmen --> EC : (interface_info, order_info, ticket_info, order_details, httpstatus)
        deactivate Kenmen
    end

    EC -> EC : 辞書の更新\nupdate()で結果をマージ
    EC --> EC : (interface_info, order_info, res_ticket_info, payback_info, order_details, httpstatus)
    deactivate EC
end

EC -> XMLDef : ec_ticket_output_xml()\n★今回修正：order_details引数追加
activate XMLDef
note right of XMLDef #ffcccc
    ★今回修正箇所
    要求区分B,H,Jの場合は
    order_detailsを含むXMLを生成
end note
XMLDef -> XMLDef : 処理結果判定
alt 処理結果エラー
    XMLDef -> XMLDef : エラー用XMLテンプレート
else 要求区分E,L (払戻し)
    XMLDef -> XMLDef : payback_info用XMLテンプレート
else 要求区分B,H,J (取消) ★今回修正
    XMLDef -> XMLDef : order_details含むXMLテンプレート ★追加
else その他
    XMLDef -> XMLDef : 通常XMLテンプレート
end
XMLDef --> EC : xml_response (Windows-31J)
deactivate XMLDef

EC -> Logger : レスポンスログ出力
EC --> Main : Response(xml_response, status, content_type)
deactivate EC

Main -> Recorder : save_response()\nレスポンス記録
Main -> Logger : レスポンス送信ログ
Main --> Client : HTTP Response\nXML形式 (Windows-31J)
deactivate Main

note over Client, Recorder #e6f3ff
    【今回の修正ポイント】
    1. 要求区分B,H,Jでorder_details辞書を生成
    2. XMLレスポンスにorder_details要素を追加
    3. 各処理モジュールでorder_details対応
end note

@enduml
```

## 概要
EC関連問合せ・発券・払戻し・完了通知のMockサーバのシーケンス図です。

### 今回の修正ポイント
- **要求区分B,H,J**（チケット発券取消、プリンタエラー発券取消、XMLファイル取得NG発券取消）で`order_details`辞書を生成
- **XMLレスポンス**に`order_details`要素を追加  
- **各処理モジュール**で`order_details`対応を実装

### 対象ファイル
- `ec_ticket.py` - order_details引数対応
- `ec_ticket_nomal.py` - 通常問合せ処理でorder_details生成追加
- `ec_ticket_complete.py` - 完了通知処理でorder_details生成追加
- `ec_ticket_kenmen.py` - 券面情報処理でorder_details生成追加
- `ec_ticket_xmldef.py` - order_details含むXMLテンプレート追加
