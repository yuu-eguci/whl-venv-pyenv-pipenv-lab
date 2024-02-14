whl-venv-pyenv-pipenv-lab
===

🐍 🐳 Laboratory for `whl`, `venv`, `pyenv`, and `pipenv`.

About whl.

- whl ってのは、それさえあれば `pip install` できるやつ。
- whl の実体は zip だから、いつもの手順で中身を見れるよ。
- whl はアーキテクチャとか Python のバージョンに依存してて、その情報はファイル名で分かるよ。
    - `{配布物}-{バージョン}(-{ビルドタグ})?-{python タグ}-{abi タグ}-{プラットフォームタグ}.whl`
- 環境に合ってない whl をインストールしようとすると、 `not a supported wheel on this platform` ってエラーが出るよ。

About venv.

- 一部の環境では `error: externally-managed-environment` が出る。
    - グローバルスコープの Python にモジュールを入れんな、仮想環境に入れろや、っていうエラー。
- venv は、 Python (3.3 以降) 標準のモジュールなので、使うのはカンタン。グローバルスコープの Python バージョンでいいんだけど、モジュールをグローバルスコープにインストールすることは避けたいなあ。仮想環境を使いたいなあ、っていうときに使う。
- pyenv は、やっぱ入れるのちょっとだけ面倒。でも Python のバージョンを切り替えたい、なんてときに便利なヤツ。
- 昔は、 venv は使い所がよくわかっていなくて、ただ pyenv とか pipenv の痒いところに手が届かないヤツ、と思っていた。でも、今日の学びで、 "バージョンを変えたいなら pyenv 使えばいいけどそのバージョンでいいなら venv でいいんじゃない?" という知見を得たよ。
- だから (?) GitHub Actions では `venv` で仮想環境を作るんだなあ。
- バージョンを変えたいわけでもないのに pyenv を使ってるときは「あれ?」って思うべき。
- lock ファイル使いたいなら (依存関係の正確なバージョンを固定したいなら) pipenv 要るけどね。
    - たとえば今回のような、 whl があるのでバージョン固定は不要、っていうときは pipenv 不要だし。

Misc.

- このノートに書いてる whl は友達に借りたものだからコミットには含めないよ。

## Struggles in Docker Container

```bash
# やっぱり Docker が最高だよな。
docker compose up -d; docker compose exec py11-service sh

python -V
# --> Python 3.11.8

python -m pip install pipenv
# --> Successfully installed

python -m pip --version
# --> pip 24.0 from /usr/local/lib/python3.11/site-packages/pip (python 3.11)

export PIPENV_VENV_IN_PROJECT=1; pipenv install

pipenv run python -m pip install owlifttypeh-1.0.1-py3-none-linux_armv7l.whl --verbose
# --> Using pip 23.3.1 from /app/.venv/lib/python3.11/site-packages/pip (python 3.11)
# --> ERROR: owlifttypeh-1.0.1-py3-none-linux_armv7l.whl is not a supported wheel on this platform.
# 再現した!

uname -m
# --> x86_64
# GPT ちゃんに、 "ARMv7 ではないからじゃね? Raspberry Pi とかで試せば?" と言われた。
# Raspberry Pi でも発生するんだけど、いま、 Docker 環境で試し続けることは不可能だな……。
```

`owlifttypeh-1.0.1-py3-none-linux_armv7l.whl` のインストールを Docker 環境で試してみたんだけど、エラー文は無事再現。

```
pipenv run python -m pip install owlifttypeh-1.0.1-py3-none-linux_armv7l.whl --verbose
# --> Using pip 23.3.1 from /app/.venv/lib/python3.11/site-packages/pip (python 3.11)
# --> ERROR: owlifttypeh-1.0.1-py3-none-linux_armv7l.whl is not a supported wheel on this platform.
```

だけどこれは……

```bash
uname -m
# --> x86_64
# GPT ちゃんに、 "ARMv7 ではないからじゃね? Raspberry Pi とかで試せば?" と言われた。
```

翌日、 Raspberry Pi で試してみた。

```bash
uname -m
# --> aarch64 (ARM アーキテクチャ 64bit)
# armv7l (ARM アーキテクチャ 32bit) ではない……。これが "not a supported wheel on this platform" の原因じゃないかな。
```

- この whl は armv7l (ARM アーキテクチャ 32bit) 向けにビルドされてる。
- その Raspberry Pi は aarch64 (ARM アーキテクチャ 64bit)。
- たぶんこれが `"not a supported wheel on this platform"` の原因。
- ドキュメントの PDF にあるよう "Raspberry Pi OS Buster 32bit" を使うとよさそう。

いちおう venv も試しておくか。

```bash
python -m venv .venv

source .venv/bin/activate

echo $VIRTUAL_ENV
# --> /app/.venv

which python
# --> /app/.venv/bin/python

python -m pip install owlifttypeh-1.0.1-py3-none-linux_armv7l.whl --verbose
# --> Using pip 24.0 from /app/.venv/lib/python3.11/site-packages/pip (python 3.11)
# --> ERROR: owlifttypeh-1.0.1-py3-none-linux_armv7l.whl is not a supported wheel on this platform.

exit

rm -rf .venv/
```

## Struggles in Raspberry Pi

```bash
python -m pip install pipenv
# --> error: externally-managed-environment
# 個人が使う Python は、個人が仮想環境 (venv) で個別にインストールして管理するべき、ということらしい。

# pipenv を入れる。
sudo apt install -y pipenv

# pyenv を入れる。
# https://github.com/pyenv/pyenv#installation
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
# https://github.com/pyenv/pyenv/wiki#suggested-build-environment
sudo apt update; sudo apt install build-essential libssl-dev zlib1g-dev \
libbz2-dev libreadline-dev libsqlite3-dev curl \
libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
# NOTE: コマンド多いなあ。
#       なるほど、まずは venv を使うことを考えて、バージョンを好きに変えるみたいなことがしたいとき pyenv を考えるのがいいのか。
#       venv は Python 標準で、 pyenv は Python のバージョンを変えるための外部ツールってことか。
#       習得したぞ。

export PIPENV_VENV_IN_PROJECT=1; pipenv install --python 3.10
# とはいえこのあと、 "whl is not a supported wheel on this platform." で詰まったんだけどな。
```
