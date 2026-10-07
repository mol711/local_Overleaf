# ベースイメージをマルチプラットフォームmanifest digestまで固定
FROM sharelatex/sharelatex:5.1.0@sha256:790b655a04ecdc07ea53276d6e3c0e5bb8cd016676402c71ea73007d2f801015

# タイムゾーンと環境変数の設定
ENV TZ=Asia/Tokyo
# sharelatex:5.1.0 に含まれる TeX Live 2024 の実行ファイルを優先する
ENV PATH=/usr/local/texlive/2024/bin/x86_64-linux:/usr/local/texlive/2024/bin/aarch64-linux:${PATH}

# TeX Live 2024 の最終更新を固定保存した公式アーカイブを使う。
# rolling な tlnet を使うと、2024 のローカルTLPDBと異なる年度の索引が混ざる。
ARG TEXLIVE_REPOSITORY=https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/2024/tlnet-final

# 必要最低限の依存パッケージと日本語フォント（Noto Sans/Serif CJK）のインストール
RUN apt-get update && apt-get install -y --no-install-recommends \
    ghostscript \
    fonts-noto-cjk \
    fonts-noto-cjk-extra \
    && rm -rf /var/lib/apt/lists/*

# TeX Live 2024 の最終状態へ更新してからフルパッケージを導入し、formatを検証する
RUN tlmgr option repository "${TEXLIVE_REPOSITORY}" && \
    tlmgr update --self --all && \
    tlmgr install scheme-full && \
    kpsewhich utf8mex.ini && \
    fmtutil-sys --all
