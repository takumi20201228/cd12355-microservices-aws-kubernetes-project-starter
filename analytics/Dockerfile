
# Python 3.10 slimイメージを使用（requirements.txtと互換性あり）

FROM python:3.10-slim



# 作業ディレクトリを設定

WORKDIR /app



# PostgreSQLクライアントライブラリのインストール（psycopg2-binaryに必要）

RUN apt-get update -y && apt-get install -y \

    build-essential \

    libpq-dev \

    && rm -rf /var/lib/apt/lists/*



# 依存関係をインストール

COPY requirements.txt .

RUN pip install --upgrade pip && \

    pip install -r requirements.txt



# アプリケーションコードをコピー

COPY . .



# ポート5153を公開

EXPOSE 5153



# アプリケーションを起動

CMD ["python", "app.py"]

