<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>高画質 YouTube ダウンローダー (単一HTML)</title>
    <style>
        * { box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            max-width: 640px;
            margin: 40px auto;
            padding: 20px;
            background: #f8f9fa;
            text-align: center;
        }
        .container {
            background: white;
            border-radius: 16px;
            padding: 30px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.08);
        }
        h1 { margin-top: 0; color: #ff0000; }
        input {
            width: 100%;
            padding: 14px 16px;
            font-size: 16px;
            border: 2px solid #ddd;
            border-radius: 10px;
            transition: 0.2s;
        }
        input:focus {
            border-color: #ff0000;
            outline: none;
        }
        button {
            width: 100%;
            padding: 14px;
            margin-top: 16px;
            font-size: 18px;
            font-weight: bold;
            background: #ff0000;
            color: white;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            transition: background 0.2s;
        }
        button:hover { background: #cc0000; }
        button:disabled {
            background: #888;
            cursor: not-allowed;
        }
        #status {
            margin-top: 20px;
            padding: 12px;
            border-radius: 8px;
            background: #f1f3f5;
            min-height: 50px;
            font-size: 14px;
            word-break: break-all;
        }
        .info {
            margin-top: 30px;
            padding-top: 20px;
            border-top: 1px solid #e9ecef;
            font-size: 13px;
            color: #666;
            text-align: left;
        }
        .info strong { color: #333; }
        .quality-badge {
            display: inline-block;
            background: #28a745;
            color: white;
            padding: 2px 10px;
            border-radius: 20px;
            font-size: 13px;
            margin-left: 8px;
        }
    </style>
</head>
<body>
<div class="container">
    <h1>📥 YouTube 高画質 DL</h1>
    <p style="color: #666;">URLを貼って「ダウンロード」を押すだけ</p>

    <input type="text" id="urlInput" placeholder="https://www.youtube.com/watch?v=..." />
    <button id="downloadBtn">⬇ ダウンロード (最高画質を自動選択)</button>
    <div id="status">📌 ここにステータスが表示されます</div>

    <div class="info">
        <p>✅ <strong>画質向上の工夫</strong>: 外部API（Piped）から取得した複数の動画ストリームを解析し、<strong>「高さ(height)」が最大のもの</strong>（1080p > 720p > 480p）を自動で選んでいます。</p>
        <p>📱 <strong>iPhoneで写真フォルダに保存するには</strong>:</p>
        <ul style="margin: 0; padding-left: 20px;">
            <li><strong>方法A</strong>: ダウンロード開始後、表示されるリンクを<strong>長押し</strong> → 「ビデオを保存」をタップ</li>
            <li><strong>方法B</strong>: ダウンロード後、「ファイル」アプリで共有ボタン → 「ビデオを保存」</li>
        </ul>
        <p style="color: #999; font-size: 12px;">⚠ このツールは非公式APIを利用しています。動画の権利者を尊重し、個人利用の範囲でお使いください。</p>
    </div>
</div>

<script>
    document.getElementById('downloadBtn').addEventListener('click', async function() {
        const urlInput = document.getElementById('urlInput');
        const status = document.getElementById('status');
        const btn = this;

        const url = urlInput.value.trim();
        if (!url) {
            alert('YouTubeのURLを入力してください。');
            return;
        }

        // YouTubeの動画IDを抽出
        const videoIdMatch = url.match(/(?:v=|\/)([0-9A-Za-z_-]{11})(?:[?&]|$)/);
        if (!videoIdMatch) {
            status.innerHTML = '❌ 有効なYouTube URLではありません。';
            return;
        }
        const videoId = videoIdMatch[1];

        // UIをロック
        btn.disabled = true;
        btn.textContent = '⏳ 情報取得中...';
        status.innerHTML = `🔍 動画ID: ${videoId} の情報を取得中...`;

        try {
            // ★ ここがポイント: CORS対応のPiped APIを使用
            const apiUrl = `https://pipedapi.kavin.rocks/streams/${videoId}`;
            const response = await fetch(apiUrl);

            if (!response.ok) {
                throw new Error(`APIエラー (HTTP ${response.status})`);
            }

            const data = await response.json();

            // 動画ストリーム（ビデオのみ）を取得し、高さ(height)で降順ソート
            const videoStreams = data.videoStreams || [];
            if (videoStreams.length === 0) {
                status.innerHTML = '❌ この動画にはダウンロード可能なストリームがありません。';
                btn.disabled = false;
                btn.textContent = '⬇ ダウンロード';
                return;
            }

            // 高画質順にソート (heightが大きい順)
            const sorted = videoStreams.sort((a, b) => (b.height || 0) - (a.height || 0));
            const best = sorted[0]; // 最高画質

            // 画質情報を表示 (例: "1080p")
            const qualityLabel = best.quality || `${best.height}p`;
            status.innerHTML = `✅ 最高画質 <strong>${qualityLabel}</strong> を取得しました。ダウンロードを開始します...`;

            // ダウンロード用のリンクを作成 (download属性で自動再生を防止)
            const downloadUrl = best.url;
            if (!downloadUrl) {
                status.innerHTML = '❌ 動画のURLが取得できませんでした。';
                btn.disabled = false;
                btn.textContent = '⬇ ダウンロード';
                return;
            }

            // aタグを生成して自動クリック (これが一番確実なダウンロード発火方法)
            const a = document.createElement('a');
            a.href = downloadUrl;
            // ファイル名に動画タイトルを入れる (オプション)
            const title = data.title ? data.title.replace(/[^a-zA-Z0-9 ]/g, '') : 'video';
            a.download = `${title}.mp4`;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);

            // 補足: もしリンクがブラウザで直接開いてしまう場合（ごく稀）に備えて、別の方法も案内
            status.innerHTML = `✅ ダウンロードが開始されました！<br> 
                                <span style="font-size:13px;color:#666;">
                                ⚠ もし動画が再生されてしまった場合（自動ダウンロード失敗）は、<br>
                                下記のリンクを <strong>右クリック / 長押し</strong> して「名前を付けてリンクを保存」してください。<br>
                                <a href="${downloadUrl}" target="_blank" style="word-break:break-all;font-size:12px;">${downloadUrl}</a>
                                </span>`;

        } catch (error) {
            console.error(error);
            status.innerHTML = `❌ エラーが発生しました: ${error.message}<br>
                                <span style="font-size:13px;color:#999;">(Piped APIが一時的にダウンしている可能性があります。時間をおいて再試行してください。)</span>`;
        } finally {
            btn.disabled = false;
            btn.textContent = '⬇ ダウンロード';
        }
    });
</script>
</body>
</html>
