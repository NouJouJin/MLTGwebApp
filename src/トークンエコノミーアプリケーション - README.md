# トークンエコノミーアプリケーション - README

このREADMEでは、提供されたReactアプリケーションの各コード行が何をしているかを詳細に説明します。アプリケーションは、ユーザーのウォレットと連携し、NFT（Non-Fungible Token）を管理・操作するためのインターフェースを提供します。

## 目次

1. [概要](#概要)
2. [インポートセクション](#インポートセクション)
3. [定数の定義](#定数の定義)
4. [ヘルパー関数](#ヘルパー関数)
    - [getAccount](#getAccount)
    - [handleAccountChanged](#handleAccountChanged)
    - [getChainName](#getChainName)
    - [getChainID](#getChainID)
    - [handleCollectonSelect](#handleCollectonSelect)
    - [handleNewContract](#handleNewContract)
    - [handleMint](#handleMint)
    - [handleLogout](#handleLogout)
    - [handleSubmit](#handleSubmit)
5. [メインコンポーネント - App](#メインコンポーネント---App)
    - [ステートの初期化](#ステートの初期化)
    - [モーダルハンドラー](#モーダルハンドラー)
    - [特定のキーの定義](#特定のキーの定義)
    - [ウォレットの初期化](#ウォレットの初期化)
    - [useEffectフック](#useEffectフック)
    - [レンダリング](#レンダリング)
6. [まとめ](#まとめ)

---

## 概要

このReactアプリケーションは、ユーザーがMetaMaskを通じてウォレットを接続し、そのウォレットに関連付けられたNFTを取得、表示、管理することを目的としています。主な機能には以下が含まれます：

- **ウォレット接続**: MetaMaskを使用してユーザーのウォレットを接続。
- **NFTの取得と表示**: Moralis APIを利用してウォレット内のNFTを取得し、一覧表示。
- **NFTの詳細表示と操作**: 選択したNFTの詳細情報を表示し、特定の操作（例：交換やミント）を実行。
- **新しいNFTコレクションの作成**: Thirdweb SDKを使用して新しいNFTコレクションをデプロイ。
- **フォームの入力と送信**: ユーザー情報を入力し、特定のアクション（例：Tシャツの交換申請）を実行。

---

## インポートセクション

```javascript
import { ThirdwebProvider } from "@thirdweb-dev/react";
import { ThirdwebSDK } from "@thirdweb-dev/sdk";
import { ethers } from "ethers";

import "./App.css";
import { useEffect, useState } from "react";

import logo from "./images/logo.png";

import "bootstrap/dist/css/bootstrap.min.css";
import Container from "react-bootstrap/esm/Container";
import Navbar from "react-bootstrap/esm/Navbar";
import Nav from "react-bootstrap/esm/Nav";
import Row from "react-bootstrap/esm/Row";
import Col from "react-bootstrap/esm/Col";
import Tab from "react-bootstrap/esm/Tab";
import Tabs from "react-bootstrap/esm/Tabs";
import Button from "react-bootstrap/esm/Button";
import Table from "react-bootstrap/esm/Table";
import Modal from "react-bootstrap/esm/Modal";
import Form from "react-bootstrap/esm/Form";
import Stack from "react-bootstrap/esm/Stack";
import OverlayTrigger from "react-bootstrap/esm/OverlayTrigger";
import Tooltip from "react-bootstrap/esm/Tooltip";
```

### 説明

- **Thirdweb関連ライブラリ**:
  - `ThirdwebProvider`, `ThirdwebSDK`: NFTやスマートコントラクトとのインタラクションに使用。
  
- **ethers**:
  - Ethereumブロックチェーンとの通信やウォレット操作に使用。

- **CSSと画像**:
  - `./App.css`: アプリケーションのスタイルシート。
  - `logo.png`: アプリのロゴ画像。

- **React Hooks**:
  - `useEffect`, `useState`: Reactの状態管理と副作用処理に使用。

- **Bootstrap関連**:
  - `bootstrap/dist/css/bootstrap.min.css`: Bootstrapのスタイルシート。
  - `Container`, `Navbar`, `Nav`, `Row`, `Col`, `Tab`, `Tabs`, `Button`, `Table`, `Modal`, `Form`, `Stack`, `OverlayTrigger`, `Tooltip`: Bootstrapのコンポーネント。

---

## 定数の定義

```javascript
const chainIdList = [
  { id: 1, name: "eth" },
  { id: 5, name: "goerli" },
  { id: 137, name: "polygon" },
  { id: 80001, name: "mumbai" },
  { id: 84532, name: "Base Sepolia" },
  { id: 8453, name: "Base" },
];
```

### 説明

- **`chainIdList`**:
  - 対応するブロックチェーンネットワークのリスト。
  - 各オブジェクトは`id`（チェーンID）と`name`（チェーン名）を持つ。

---

## ヘルパー関数

### getAccount

```javascript
const getAccount = async () => {
  try {
    const account = await window.ethereum.request({ method: "eth_requestAccounts" });
    if (account.length > 0) {
      return account[0];
    } else {
      return "";
    }
  } catch (err) {
    if (err.code === 4001) {
      // EIP-1193 userRejectedRequest error
      // If this happens, the user rejected the connection request.
      console.log("Please connect to MetaMask.");
    } else {
      console.error(err);
    }
    return "";
  }
};
```

### 説明

- **`getAccount`関数**:
  - ユーザーのウォレットアカウントを取得。
  - MetaMaskに接続を要求し、接続が成功すれば最初のアカウントアドレスを返す。
  - ユーザーが接続を拒否した場合やエラーが発生した場合は空文字列を返す。

---

### handleAccountChanged

```javascript
const handleAccountChanged = async (accountNo, setAccount, setChainId, setNfts, setCollections, setChainName) => {
  const account = await getAccount();
  setAccount(account); //walletID

  const chainId = await getChainID();
  setChainId(chainId); //メタマスクで設定されているチェーン

  const chainName = await getChainName(chainId);
  setChainName(chainName);

  console.log(process.env.WEB3_API_KEY); //thirdwebのAPIキー
  const web3ApiKey = process.env.REACT_APP_MORALIS_API_KEY;
  const options = {
    method: "GET",
    headers: {
      accept: "application/json",
      "X-API-Key": web3ApiKey,
    },
  };

  const resNftData = await fetch(`https://deep-index.moralis.io/api/v2.2/${account}/nft?chain=${chainName}`, options); //v2.2に更新
  const resNft = await resNftData.json();
  console.log(JSON.stringify(resNft));

  let nfts = [];
  for (let nft of resNft.result) {
    const tmp = JSON.parse(nft.metadata);
    console.log(JSON.stringify(tmp));
    if (tmp !== null) {
      const optionTokenInfo = {
        method: "GET",
        headers: {
          // Authorization: process.env.AIRTABLE_API_KEY, //AirTableのAPIキー
          Authorization: "Bearer pat03Tla2ix8YDjUG.fb25c1823ec93914194e26699728d0c10923bdb68aa71fc7301a059bdf5990d7",//AirTableのAPIキー書き換え（旧：keypjfrOALL1xCF3r）
        },
      };
      const resTokenInfo = await fetch(
        //airTableのNFTのaddressとIDを取得している
        `https://api.airtable.com/v0/appq0R9tJ2BkvKhRt/tblGeRuC0iRypjYfl?filterByFormula=AND(%7BContract_ID%7D%3D%22${nft.token_address}%22%2C%7BToken_ID%7D%3D${nft.token_id})`,
        optionTokenInfo
      );
      const resTokenInfoJson = await resTokenInfo.json();
      if (resTokenInfoJson.records[0] !== undefined) {
        console.log(JSON.stringify(resTokenInfoJson.records[0]));
        const nftinfo = {
          contract_name: nft.name,
          image: tmp.image !== "" ? `https://ipfs.io/ipfs/${tmp.image.substring(7)}` : "",
          nft_name: tmp.name,
          present_detail: resTokenInfoJson.records[0].fields.Thanks_Gift,
          token_address: nft.token_address,
          token_id: nft.token_id,
          amount: nft.amount,
          key_id: resTokenInfoJson.records[0].fields.Key_ID,
        };
        nfts.push(nftinfo);
      }
    }
  }

  //発行日でソート
  setNfts(
    nfts.sort((a, b) => {
      var r = 0;
      if (a.issue_date > b.issue_date) {
        r = -1;
      } else if (a.issue_date < b.issue_date) {
        r = 1;
      }
      return r;
    })
  );
};
```

### 説明

- **`handleAccountChanged`関数**:
  - アカウントやチェーンが変更された際に呼び出される。
  - 現在のアカウントを取得し、ステートを更新。
  - 現在のチェーンIDとチェーン名を取得し、ステートを更新。
  - Moralis APIを使用してユーザーのNFTを取得。
  - 各NFTのメタデータを解析し、AirTable APIを使用して追加情報（`Thanks_Gift`や`Key_ID`）を取得。
  - 取得したNFT情報を発行日でソートし、ステートを更新。

---

### getChainName

```javascript
const getChainName = async (chainId) => {
  let data = chainIdList.filter(function (item) {
    return item.id === chainId;
  });

  return data[0].name;
};
```

### 説明

- **`getChainName`関数**:
  - 指定された`chainId`に対応するチェーン名を`chainIdList`から取得。
  - 該当するチェーンが存在する場合、その名前を返す。

---

### getChainID

```javascript
const getChainID = async () => {
  const chainId = await window.ethereum.request({ method: "eth_chainId" });
  return parseInt(chainId);
};
```

### 説明

- **`getChainID`関数**:
  - MetaMaskから現在のチェーンIDを取得。
  - 取得したチェーンIDを整数に変換して返す。

---

### handleCollectonSelect

```javascript
const handleCollectonSelect = async (chainName, setSelectedCollection, setSelectedCollectionName, setMintedNfts) => {
  let selectedCollection = "";
  let elements = document.getElementsByName("collections");
  for (let i in elements) {
    if (elements.item(i).checked) {
      selectedCollection = elements.item(i).id;
      setSelectedCollection(selectedCollection);
      setSelectedCollectionName(elements.item(i).value);
    }
  }

  const web3ApiKey = "27XAH0PFnvHnMN7EgbXAQiQH5ycsAuE3dduoJVtE5EQFwVklhnFTebDxlAiihvgV";
  const options = {
    method: "GET",
    headers: {
      accept: "application/json",
      "X-API-Key": web3ApiKey,
    },
  };

  const resNftData = await fetch(`https://deep-index.moralis.io/api/v2.2/nft/${selectedCollection}?chain=${chainName}&format=decimal`, options); //v2.2に更新
  const resNft = await resNftData.json();
  let nfts = [];
  for (let nft of resNft.result) {
    const tmp = JSON.parse(nft.metadata);
    if (tmp !== null) {
      if ("attributes" in tmp) {
        let issue_date = "";
        let issuer_name = "";
        let owner_address = "";
        let genre = "";
        for (const attribute of tmp.attributes) {
          if (attribute.trait_type === "exp_type") {
            genre = attribute.value;
          } else if (attribute.trait_type === "ca_name" || attribute.trait_type === "issuer_name") {
            issuer_name = attribute.value;
          } else if (attribute.trait_type === "owner_address") {
            owner_address = attribute.value;
          } else if (attribute.trait_type === "cert_date" || attribute.trait_type === "issue_date") {
            issue_date = attribute.value.substring(0, 4) + "/" + attribute.value.substring(4, 6) + "/" + attribute.value.substring(6);
          }
        }

        const nftinfo = {
          name: tmp.name,
          image: tmp.image !== "" ? `https://ipfs.io/ipfs/${tmp.image.substring(7)}` : "",
          issue_date: issue_date,
          issuer_name: issuer_name,
          owner_address: owner_address,
          genre: genre,
          description: tmp.description,
          token_address: nft.token_address,
        };

        nfts.push(nftinfo);
      }
    }
  }
  setMintedNfts(
    nfts.sort((a, b) => {
      var r = 0;
      if (a.issue_date > b.issue_date) {
        r = -1;
      } else if (a.issue_date < b.issue_date) {
        r = 1;
      }

      return r;
    })
  );
};
```

### 説明

- **`handleCollectonSelect`関数**:
  - ユーザーが選択したNFTコレクションを取得。
  - 選択されたコレクションのIDと名前をステートに設定。
  - Moralis APIを使用して選択されたコレクションのNFTを取得。
  - 各NFTのメタデータを解析し、発行日、発行者名、所有者アドレス、ジャンルなどの情報を抽出。
  - 抽出したNFT情報を発行日でソートし、ステートを更新。

---

### handleNewContract

```javascript
const handleNewContract = async (account, chainName, setDisable, setCollections, setShowNewToken) => {
  setDisable(true);

  let cn = chainName;
  if (chainName === "eth") {
    cn = "mainnet";
  }

  const provider = new ethers.providers.Web3Provider(window.ethereum);
  await provider.send("eth_requestAccounts", []);
  const signer = await provider.getSigner();
  const sdk = ThirdwebSDK.fromSigner(signer, cn);

  const contractAddress = await sdk.deployer.deployNFTCollection({
    name: document.getElementById("token_name").value,
    symbol: document.getElementById("token_symbol").value,
    primary_sale_recipient: account,
  });

  const metadata = {
    name: "First NFT",
    description: "First NFT to show in Q list.",
    image: "",
  };

  const contract = await sdk.getContract(contractAddress);
  await contract.erc721.mint(metadata);

  setDisable(false);
  setShowNewToken(false);

  document.getElementById("reloadContract").click();
};
```

### 説明

- **`handleNewContract`関数**:
  - 新しいNFTコレクションをデプロイ。
  - フォームから取得したトークン名とシンボルを使用。
  - デプロイ後、初回のNFTをミント。
  - デプロイ中はボタンを無効化し、完了後に再度有効化。
  - 新しいコントラクトの読み込みをトリガー。

---

### handleMint

```javascript
const handleMint = async (selectedCollection, chainName, setDisable, setMintedNfts, setShow) => {
  setDisable(true);

  let cn = chainName;
  if (chainName === "eth") {
    cn = "mainnet";
  }

  const account = await getAccount();
  const issue_date = document.getElementById("issue_date").value.replace(/[^0-9]/g, "");
  const owner = document.getElementById("owner").value;
  const title = document.getElementById("exp_type").value;
  const issuer_name = document.getElementById("issuer_name").value;
  const exp_type = document.getElementById("exp_type").value;
  const description = document.getElementById("description").value;

  const provider = new ethers.providers.Web3Provider(window.ethereum);
  await provider.send("eth_requestAccounts", []);
  const signer = await provider.getSigner();

  const sdk = ThirdwebSDK.fromSigner(signer, cn);
  const contract = await sdk.getContract(selectedCollection);

  const walletAddress = owner;
  const metadata = {
    name: title,
    description: description,
    image: document.getElementById("image").files[0],
    attributes: [
      { trait_type: "issue_date", value: issue_date },
      { trait_type: "exp_type", value: exp_type },
      { trait_type: "issuer_address", value: account },
      { trait_type: "issuer_name", value: issuer_name },
      { trait_type: "owner_address", value: owner },
    ],
  };
  await contract.erc721.mintTo(walletAddress, metadata);

  const url = "https://7iqg4cc3ca2nuy6oqrjchnuige0rqzcu.lambda-url.ap-northeast-1.on.aws/";
  const method = "POST";
  const submitBody = {
    to: document.getElementById("email").value,
    from: issuer_name,
    chainName: cn,
    title: title,
    description: description,
    owner_address: owner,
  };
  const body = JSON.stringify(submitBody);

  fetch(url, { method, body })
    .then((res) => {
      console.log(res.status);
      if (res.ok) {
        return res.json().then((resJson) => {
          setDisable(false);
          setShow(false);

          document.getElementById("reloadCollection").click();
        });
      }
    })
    .catch((error) => {
      console.log(error);
    });
};
```

### 説明

- **`handleMint`関数**:
  - 選択されたNFTコレクションに対して新しいNFTをミント。
  - フォームからユーザー入力（発行日、所有者、タイトル、発行者名、説明、画像）を取得。
  - Thirdweb SDKを使用して指定されたコレクションに対してNFTをミント。
  - ミント後、指定のURLにPOSTリクエストを送信（おそらく通知やデータ保存のため）。
  - 処理中はボタンを無効化し、完了後に再度有効化。

**注意**: この関数は現在「未使用」とされています。

---

### handleLogout

```javascript
const handleLogout = async () => {
  window.location.href = "/";
};
```

### 説明

- **`handleLogout`関数**:
  - ユーザーをホームページ（"/"）にリダイレクトしてログアウトを実現。

---

### handleSubmit

```javascript
const handleSubmit = async (account, nft, chainName, size, otherSize, handleClose, setIsSubmitting) => {
  let cn = chainName;
  if (chainName === "eth") {
    cn = "mainnet";
  }
  const provider = new ethers.providers.Web3Provider(window.ethereum);
  await provider.send("eth_requestAccounts", []);
  const signer = await provider.getSigner();

  const sdk = ThirdwebSDK.fromSigner(signer, cn);
  const contract = await sdk.getContract(nft.token_address);

  const walletAddress = "0x6D8Dd5Cf6fa8DB2be08845b1380e886BFAb03E07";

  const amount = 1;

  const tokenId = nft.token_id;

  const name = document.getElementById("Name").value;
  const zipCode = document.getElementById("Zip_Code").value;
  const address = document.getElementById("Address").value;
  const tel = document.getElementById("Tel").value;
  const mail = document.getElementById("Mail").value;
  // const otherSizeElement = document.getElementById("Other-size");
  // const otherSize = otherSizeElement ? otherSizeElement.value : "";

  // 確認ダイアログのメッセージ
  let confirmMessage = `
  以下の情報で送信してもよろしいですか？
  
  名前: ${name}
  郵便番号: ${zipCode}
  住所: ${address}
  電話番号: ${tel}
  メール: ${mail}\n`;
  if (size) confirmMessage += `  サイズ: ${size}\n`;
  if (otherSize) confirmMessage += `  その他サイズ: ${otherSize}\n`;

  // 確認ダイアログを表示
  if (!window.confirm(confirmMessage)) {
    // ユーザーがキャンセルを選択した場合、処理を中断
    handleClose();
    setIsSubmitting(false); // 送信状態を解除
    return;
  }

  setIsSubmitting(true); // 送信状態を開始

  try {
    await contract.erc1155.transfer(walletAddress, tokenId, amount);

    const headers = {
      Authorization: "Bearer pat03Tla2ix8YDjUG.fb25c1823ec93914194e26699728d0c10923bdb68aa71fc7301a059bdf5990d7", //AirTableのAPIキー//AirTableのAPIキー書き換え（旧：keypjfrOALL1xCF3r）
      "Content-Type": "application/json",
    };
    const method = "POST";
    const submitBody = {
      records: [
        {
          fields: {
            Key_ID: nft.key_id,
            Thanks_Gift: nft.present_detail,
            Name: name,
            Zip_Code: zipCode,
            Address: address,
            Tel: tel,
            Mail: mail,
            Size: size,
            Size_Other: otherSize,
          },
        },
      ],
    };

    const body = JSON.stringify(submitBody);
    console.log(body);
    const resTokenInfo = await fetch(`https://api.airtable.com/v0/appq0R9tJ2BkvKhRt/tbld2laNlKCi7B2GW`, { method, headers, body });
    document.getElementById("GetAccountButton").click();
    alert(`${nft.nft_name}の交換完了しました。到着するまでお楽しみに！`);
  } catch (error) {
    console.error(error);
    setIsSubmitting(false); // 送信状態を解除
  }
  setIsSubmitting(false); // 送信状態を解除
  handleClose(); // モーダルを閉じる

  // 送信後にフォームをリセット
  document.getElementById("Name").value = "";
  document.getElementById("Zip_Code").value = "";
  document.getElementById("Address").value = "";
  document.getElementById("Tel").value = "";
  document.getElementById("Mail").value = "";
  document.querySelectorAll('input[type="radio"]').forEach((radio) => {
    radio.checked = false;
  });
  if (document.getElementById("Other-size")) {
    document.getElementById("Other-size").value = "";
  }
};
```

### 説明

- **`handleSubmit`関数**:
  - ユーザーがフォームを送信した際に呼び出される。
  - ユーザーから入力された情報（名前、郵便番号、住所、電話番号、メール、サイズなど）を取得。
  - 確認ダイアログを表示し、ユーザーの確認を得る。
  - 確認後、指定のウォレットアドレスに対してNFTを転送（交換）する。
  - 転送後、AirTableにユーザー情報と交換情報を送信。
  - 処理中は送信ボタンを無効化し、完了後に再度有効化。
  - 処理が完了したらフォームをリセット。

---

## メインコンポーネント - App

```javascript
function App() {
  // ステートの初期化
  const [account, setAccount] = useState("");
  const [chainId, setChainId] = useState(0);
  const [chainName, setChainName] = useState("");
  // const [index, setIndex] = useState(0);
  const [disable, setDisable] = useState(false);
  const [show, setShow] = useState(false);
  const [showNewToken, setShowNewToken] = useState(false);
  const [showDetail, setShowDetail] = useState(false);
  const [nfts, setNfts] = useState([]);
  const [selectedNft, setSelectedNft] = useState({});
  const [collections, setCollections] = useState([]);
  const [mintedNfts, setMintedNfts] = useState([]);
  const [selectedCollection, setSelectedCollection] = useState("");
  const [selectedCollectionName, setSelectedCollectionName] = useState("");
  const [selectedGiftNft, setSelectedGiftNft] = useState("");
  const location = window.location.pathname.toLowerCase();
  const [isSubmitting, setIsSubmitting] = useState(false);

  // モーダルハンドラー
  const handleClose = () => {
    size = ""; // サイズをリセット
    otherSize = ""; // その他のサイズをリセット
    setShow(false);
    setIsSubmitting(false); // 送信状態を解除
  };
  const handleShow = () => setShow(true);
  const handleCloseNewToken = () => setShowNewToken(false);
  const handleShowNewToken = () => setShowNewToken(true);
  const handleCloseDetail = () => setShowDetail(false);
  const handleShowDetail = async (nft) => {
    setSelectedNft(nft);
    // ダイアログの内容をリセット
    setSize("");
    setOtherSize("");
    // 他のダイアログ関連の状態もリセット
    setShowDetail(true);
  };

  const [size, setSize] = useState(""); // デフォルトのサイズを設定
  const [otherSize, setOtherSize] = useState(""); // 「その他」のサイズを設定
  const handleRadioChange = (event) => {
    setSize(event.target.value);
    if (event.target.value !== "その他") {
      setOtherSize(""); // 「その他」以外が選択されたら、テキストフィールドをクリア
    }
  };

  const handleOtherSizeChange = (event) => {
    setOtherSize(event.target.value);
  };

  const specificKeys = ["rec65kFu48ut5GPhC", "recB1VbiT6bR7TMnH", "recqCurt5f435BcVf", "recj2JF2UnJU2ixXw", "reclz4Dg5QS8VnJZ0", "recyBnzU9IzYtJuCT", "recK0sK8Hzq6ffghW", "recs6Mdq8UFgEjpPD", "recjmIXtdNbAdMQnd", "rec5noihUnn4JYDeV", "recCafjYmJ2gX9K0l", "reclxuPXKWLkCjmVE"]; //オリジナルTシャツのNFTのkey
  // const handleSelect = (selectedIndex, e) => {
  //   setIndex(selectedIndex);
  // };

  // ウォレットの初期化
  const initializeAccount = async () => {
    const account = getAccount();
    if (account !== "") {
      await handleAccountChanged(account, setAccount, setChainId, setNfts, setCollections, setChainName);
    }
  };

  // useEffectフック
  useEffect(() => {
    if (typeof window.ethereum !== "undefined") {
      window.ethereum.on("accountsChanged", (accountNo) => handleAccountChanged(accountNo, setAccount, setChainId, setNfts, setCollections, setChainName));
      window.ethereum.on("chainChanged", (accountNo) => handleAccountChanged(accountNo, setAccount, setChainId, setNfts, setCollections, setChainName));
    } else {
      window.addEventListener("ethereum#initialized", initializeAccount, {
        once: true,
      });

      setTimeout(initializeAccount, 3000); // 3 seconds
    }
  }, [account]);

  // レンダリング
  return (
    <div className="App d-flex flex-column">
      <div className="mb-auto w-100">
        <>
          <Navbar>
            <Container>
              <Navbar.Brand href="#home">
                <img src={logo} width="250" />
              </Navbar.Brand>
              <Navbar.Toggle />
              <Navbar.Collapse className="justify-content-end">
                <Navbar.Text>
                  <Button className="py-2 px-4 btn-lg" variant="outline-dark" id="GetAccountButton" onClick={initializeAccount}>
                    MetaMaskに接続
                  </Button>
                </Navbar.Text>
              </Navbar.Collapse>
            </Container>
          </Navbar>
          <Container className="my-1 p-1">
            <Navbar expand="lg">
              <Container className="mx-0 px-0">
                <Nav>
                  <Stack className="left-align">
                    <div>
                      <h3>
                        保有NFT一覧
                        <Button id="reloadNft" className="mb-1" variant="text" onClick={() => handleAccountChanged(account, setAccount, setChainId, setNfts, setCollections, setChainName)}>
                          <svg xmlns="http://www.w3.org/2000/svg" width="25" height="25" fill="currentColor" className="bi bi-arrow-clockwise" viewBox="0 0 16 16">
                            <path fillRule="evenodd" d="M8 3a5 5 0 1 0 4.546 2.914.5.5 0 0 1 .908-.417A6 6 0 1 1 8 2v1z" />
                            <path d="M8 4.466V.534a.25.25 0 0 1 .41-.192l2.36 1.966c.12.1.12.284 0 .384L8.41 4.658A.25.25 0 0 1 8 4.466z" />
                          </svg>
                        </Button>
                      </h3>
                    </div>
                    <div>
                      <p>
                        <small>このウォレットにある「MetagriLabo Thanks Gift（MLTG）」の一覧です。</small>
                      </p>
                    </div>
                  </Stack>
                </Nav>
              </Container>
            </Navbar>
            <Table className="table-hover" responsive={true}>
              <thead className="table-secondary">
                <tr>
                  <th>画像</th>
                  {/* <th>ID</th> */}
                  <th>コントラクト</th>
                  <th>NFT名</th>
                  <th>もらえるもの</th>
                  <th>数量</th>
                  <th>引き換え</th>
                </tr>
              </thead>
              <tbody>
                {nfts.length !== 0 ? (
                  nfts.map((nft, index) => {
                    return (
                      <tr key={index} className="align-middle">
                        <td>
                          {nft.image !== "" ? (
                            <a href={nft.image} target="_blank" rel="noreferrer">
                              <img src={nft.image} alt="nftimage" width="70px" />
                            </a>
                          ) : (
                            <></>
                          )}
                        </td>
                        {/* <td>{nft.token_id}</td> */}
                        {/* <td>{nft.key_id}</td> */}
                        <td>{nft.contract_name}</td>
                        <td>{nft.nft_name}</td>
                        <td>{nft.present_detail}</td>
                        <td>{nft.amount}</td>
                        <td>
                          <input class="form-check-input" type="radio" name="flexRadioDefault" id={index} onClick={() => handleShowDetail(nft)} />
                        </td>
                      </tr>
                    );
                  })
                ) : (
                  <></>
                )}
              </tbody>
            </Table>
            {/* 選択したNFTによって変わる */}
            {/* オリジナルTシャツ	*/}
            {selectedNft && specificKeys.includes(selectedNft.key_id) && (
              <Form>
                <Form.Group className="mb-3">
                  <Form.Label>お名前</Form.Label>
                  <Form.Control id="Name" type="text" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>郵便番号</Form.Label>
                  <Form.Control id="Zip_Code" type="text" placeholder="000-0000" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>ご住所</Form.Label>
                  <Form.Control id="Address" type="text" placeholder="都道府県市町村番地建物" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>電話番号</Form.Label>
                  <Form.Control id="Tel" type="text" placeholder="000-0000-0000" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>通知先メールアドレス</Form.Label>
                  <Form.Control id="Mail" type="email" placeholder="experience@metagri-labo.com" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>サイズ</Form.Label>
                </Form.Group>
                <Form.Group className="mb-3">
                  <div className="d-inline-block me-2">
                    <Form.Check type="radio" label="S" name="size" value="S" id="S-size" onChange={handleRadioChange} />
                  </div>

                  <div className="d-inline-block me-2">
                    <Form.Check type="radio" label="M" name="size" value="M" id="M-size" onChange={handleRadioChange} />
                  </div>

                  <div className="d-inline-block me-2">
                    <Form.Check type="radio" label="L" name="size" value="L" id="L-size" onChange={handleRadioChange} />
                  </div>
                  <div className="d-inline-block me-2">
                    <Form.Check type="radio" label="XL" name="size" value="XL" id="XL-size" onChange={handleRadioChange} />
                  </div>
                  <div className="d-inline-block me-2">
                    <Form.Check type="radio" label="その他" name="size" value="その他" id="other" onChange={handleRadioChange} />
                  </div>
                </Form.Group>

                {/* その他が選択された場合のテキストフィールド */}
                {size === "その他" && (
                  <Form.Group className="mb-3">
                    <Form.Label>その他サイズ</Form.Label>
                    <Form.Control type="text" id="Other-size" value={otherSize} onChange={handleOtherSizeChange} />
                  </Form.Group>
                )}
              </Form>
            )}
            {/* デフォルト */}
            {selectedNft && !specificKeys.includes(selectedNft.key_id) && (
              <Form>
                <Form.Group className="mb-3">
                  <Form.Label>お名前</Form.Label>
                  <Form.Control id="Name" type="text" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>郵便番号</Form.Label>
                  <Form.Control id="Zip_Code" type="text" placeholder="000-0000" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>ご住所</Form.Label>
                  <Form.Control id="Address" type="text" placeholder="都道府県市町村番地建物" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>電話番号</Form.Label>
                  <Form.Control id="Tel" type="text" placeholder="000-0000-0000" />
                </Form.Group>
                <Form.Group className="mb-3">
                  <Form.Label>通知先メールアドレス</Form.Label>
                  <Form.Control id="Mail" type="email" placeholder="experience@metagri-labo.com" />
                </Form.Group>
              </Form>
            )}
            {/* </Form> */}
            {disable === false ? (
              <>
                {/* <Button className="px-4" variant="outline-dark" onClick={handleClose}>
                  キャンセル
                </Button> */}
                <Button className="px-4" variant="outline-dark" disabled={isSubmitting} onClick={() => handleSubmit(account, selectedNft, chainName, size, otherSize, handleClose, setIsSubmitting)}>
                  申し込む
                </Button>
              </>
            ) : (
              <>
                {/* <Button className="px-4" variant="outline-dark" disabled={true}>
                  キャンセル
                </Button> */}
                <Button className="px-4" variant="dark" disabled={isSubmitting}>
                  {isSubmitting && <span className="me-2 spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>}
                  {isSubmitting ? "数分かかる場合があります..." : "申し込む"}
                </Button>
              </>
            )}
          </Container>
        </>
      </div>

      <footer className="mt-auto p-3">Ideated by Studymeter Inc.</footer>
    </div>
  );
}

export default App;
```

### 説明

#### ステートの初期化

```javascript
const [account, setAccount] = useState("");
const [chainId, setChainId] = useState(0);
const [chainName, setChainName] = useState("");
// const [index, setIndex] = useState(0);
const [disable, setDisable] = useState(false);
const [show, setShow] = useState(false);
const [showNewToken, setShowNewToken] = useState(false);
const [showDetail, setShowDetail] = useState(false);
const [nfts, setNfts] = useState([]);
const [selectedNft, setSelectedNft] = useState({});
const [collections, setCollections] = useState([]);
const [mintedNfts, setMintedNfts] = useState([]);
const [selectedCollection, setSelectedCollection] = useState("");
const [selectedCollectionName, setSelectedCollectionName] = useState("");
const [selectedGiftNft, setSelectedGiftNft] = useState("");
const location = window.location.pathname.toLowerCase();
const [isSubmitting, setIsSubmitting] = useState(false);
```

- 各種ステートを初期化。主にウォレット情報、NFTリスト、フォームの表示状態などを管理。

#### モーダルハンドラー

```javascript
const handleClose = () => {
  size = ""; // サイズをリセット
  otherSize = ""; // その他のサイズをリセット
  setShow(false);
  setIsSubmitting(false); // 送信状態を解除
};
const handleShow = () => setShow(true);
const handleCloseNewToken = () => setShowNewToken(false);
const handleShowNewToken = () => setShowNewToken(true);
const handleCloseDetail = () => setShowDetail(false);
const handleShowDetail = async (nft) => {
  setSelectedNft(nft);
  // ダイアログの内容をリセット
  setSize("");
  setOtherSize("");
  // 他のダイアログ関連の状態もリセット
  setShowDetail(true);
};
```

- 各種モーダル（フォームや詳細表示）の表示・非表示を制御するハンドラー。

#### 特定のキーの定義

```javascript
const specificKeys = ["rec65kFu48ut5GPhC", "recB1VbiT6bR7TMnH", "recqCurt5f435BcVf", "recj2JF2UnJU2ixXw", "reclz4Dg5QS8VnJZ0", "recyBnzU9IzYtJuCT", "recK0sK8Hzq6ffghW", "recs6Mdq8UFgEjpPD", "recjmIXtdNbAdMQnd", "rec5noihUnn4JYDeV", "recCafjYmJ2gX9K0l", "reclxuPXKWLkCjmVE"]; //オリジナルTシャツのNFTのkey
```

- 特定のNFTキーのリスト。オリジナルTシャツに関連するNFTを識別するために使用。

#### ウォレットの初期化

```javascript
const initializeAccount = async () => {
  const account = getAccount();
  if (account !== "") {
    await handleAccountChanged(account, setAccount, setChainId, setNfts, setCollections, setChainName);
  }
};
```

- アカウントを取得し、ウォレットの状態を初期化する関数。

#### useEffectフック

```javascript
useEffect(() => {
  if (typeof window.ethereum !== "undefined") {
    window.ethereum.on("accountsChanged", (accountNo) => handleAccountChanged(accountNo, setAccount, setChainId, setNfts, setCollections, setChainName));
    window.ethereum.on("chainChanged", (accountNo) => handleAccountChanged(accountNo, setAccount, setChainId, setNfts, setCollections, setChainName));
  } else {
    window.addEventListener("ethereum#initialized", initializeAccount, {
      once: true,
    });

    setTimeout(initializeAccount, 3000); // 3 seconds
  }
}, [account]);
```

- アプリの初期化時にウォレットの変更イベント（アカウント変更やチェーン変更）をリスン。
- `window.ethereum`が存在しない場合、`ethereum#initialized`イベントをリッスンし、3秒後にウォレットを初期化。

#### レンダリング

```javascript
return (
  <div className="App d-flex flex-column">
    <div className="mb-auto w-100">
      <>
        <Navbar>
          <Container>
            <Navbar.Brand href="#home">
              <img src={logo} width="250" />
            </Navbar.Brand>
            <Navbar.Toggle />
            <Navbar.Collapse className="justify-content-end">
              <Navbar.Text>
                <Button className="py-2 px-4 btn-lg" variant="outline-dark" id="GetAccountButton" onClick={initializeAccount}>
                  MetaMaskに接続
                </Button>
              </Navbar.Text>
            </Navbar.Collapse>
          </Container>
        </Navbar>
        <Container className="my-1 p-1">
          <Navbar expand="lg">
            <Container className="mx-0 px-0">
              <Nav>
                <Stack className="left-align">
                  <div>
                    <h3>
                      保有NFT一覧
                      <Button id="reloadNft" className="mb-1" variant="text" onClick={() => handleAccountChanged(account, setAccount, setChainId, setNfts, setCollections, setChainName)}>
                        <svg xmlns="http://www.w3.org/2000/svg" width="25" height="25" fill="currentColor" className="bi bi-arrow-clockwise" viewBox="0 0 16 16">
                          <path fillRule="evenodd" d="M8 3a5 5 0 1 0 4.546 2.914.5.5 0 0 1 .908-.417A6 6 0 1 1 8 2v1z" />
                          <path d="M8 4.466V.534a.25.25 0 0 1 .41-.192l2.36 1.966c.12.1.12.284 0 .384L8.41 4.658A.25.25 0 0 1 8 4.466z" />
                        </svg>
                      </Button>
                    </h3>
                  </div>
                  <div>
                    <p>
                      <small>このウォレットにある「MetagriLabo Thanks Gift（MLTG）」の一覧です。</small>
                    </p>
                  </div>
                </Stack>
              </Nav>
            </Container>
          </Navbar>
          <Table className="table-hover" responsive={true}>
            <thead className="table-secondary">
              <tr>
                <th>画像</th>
                {/* <th>ID</th> */}
                <th>コントラクト</th>
                <th>NFT名</th>
                <th>もらえるもの</th>
                <th>数量</th>
                <th>引き換え</th>
              </tr>
            </thead>
            <tbody>
              {nfts.length !== 0 ? (
                nfts.map((nft, index) => {
                  return (
                    <tr key={index} className="align-middle">
                      <td>
                        {nft.image !== "" ? (
                          <a href={nft.image} target="_blank" rel="noreferrer">
                            <img src={nft.image} alt="nftimage" width="70px" />
                          </a>
                        ) : (
                          <></>
                        )}
                      </td>
                      {/* <td>{nft.token_id}</td> */}
                      {/* <td>{nft.key_id}</td> */}
                      <td>{nft.contract_name}</td>
                      <td>{nft.nft_name}</td>
                      <td>{nft.present_detail}</td>
                      <td>{nft.amount}</td>
                      <td>
                        <input class="form-check-input" type="radio" name="flexRadioDefault" id={index} onClick={() => handleShowDetail(nft)} />
                      </td>
                    </tr>
                  );
                })
              ) : (
                <></>
              )}
            </tbody>
          </Table>
          {/* 選択したNFTによって変わる */}
          {/* オリジナルTシャツ	*/}
          {selectedNft && specificKeys.includes(selectedNft.key_id) && (
            <Form>
              <Form.Group className="mb-3">
                <Form.Label>お名前</Form.Label>
                <Form.Control id="Name" type="text" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>郵便番号</Form.Label>
                <Form.Control id="Zip_Code" type="text" placeholder="000-0000" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>ご住所</Form.Label>
                <Form.Control id="Address" type="text" placeholder="都道府県市町村番地建物" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>電話番号</Form.Label>
                <Form.Control id="Tel" type="text" placeholder="000-0000-0000" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>通知先メールアドレス</Form.Label>
                <Form.Control id="Mail" type="email" placeholder="experience@metagri-labo.com" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>サイズ</Form.Label>
              </Form.Group>
              <Form.Group className="mb-3">
                <div className="d-inline-block me-2">
                  <Form.Check type="radio" label="S" name="size" value="S" id="S-size" onChange={handleRadioChange} />
                </div>

                <div className="d-inline-block me-2">
                  <Form.Check type="radio" label="M" name="size" value="M" id="M-size" onChange={handleRadioChange} />
                </div>

                <div className="d-inline-block me-2">
                  <Form.Check type="radio" label="L" name="size" value="L" id="L-size" onChange={handleRadioChange} />
                </div>
                <div className="d-inline-block me-2">
                  <Form.Check type="radio" label="XL" name="size" value="XL" id="XL-size" onChange={handleRadioChange} />
                </div>
                <div className="d-inline-block me-2">
                  <Form.Check type="radio" label="その他" name="size" value="その他" id="other" onChange={handleRadioChange} />
                </div>
              </Form.Group>

              {/* その他が選択された場合のテキストフィールド */}
              {size === "その他" && (
                <Form.Group className="mb-3">
                  <Form.Label>その他サイズ</Form.Label>
                  <Form.Control type="text" id="Other-size" value={otherSize} onChange={handleOtherSizeChange} />
                </Form.Group>
              )}
            </Form>
          )}
          {/* デフォルト */}
          {selectedNft && !specificKeys.includes(selectedNft.key_id) && (
            <Form>
              <Form.Group className="mb-3">
                <Form.Label>お名前</Form.Label>
                <Form.Control id="Name" type="text" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>郵便番号</Form.Label>
                <Form.Control id="Zip_Code" type="text" placeholder="000-0000" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>ご住所</Form.Label>
                <Form.Control id="Address" type="text" placeholder="都道府県市町村番地建物" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>電話番号</Form.Label>
                <Form.Control id="Tel" type="text" placeholder="000-0000-0000" />
              </Form.Group>
              <Form.Group className="mb-3">
                <Form.Label>通知先メールアドレス</Form.Label>
                <Form.Control id="Mail" type="email" placeholder="experience@metagri-labo.com" />
              </Form.Group>
            </Form>
          )}
          {/* </Form> */}
          {disable === false ? (
            <>
              {/* <Button className="px-4" variant="outline-dark" onClick={handleClose}>
                キャンセル
              </Button> */}
              <Button className="px-4" variant="outline-dark" disabled={isSubmitting} onClick={() => handleSubmit(account, selectedNft, chainName, size, otherSize, handleClose, setIsSubmitting)}>
                申し込む
              </Button>
            </>
          ) : (
            <>
              {/* <Button className="px-4" variant="outline-dark" disabled={true}>
                キャンセル
              </Button> */}
              <Button className="px-4" variant="dark" disabled={isSubmitting}>
                {isSubmitting && <span className="me-2 spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>}
                {isSubmitting ? "数分かかる場合があります..." : "申し込む"}
              </Button>
            </>
          )}
        </Container>
      </>
    </div>

    <footer className="mt-auto p-3">Ideated by Studymeter Inc.</footer>
  </div>
);
```

### 説明

#### ステートの初期化

- **アカウント情報**:
  - `account`, `setAccount`: ユーザーのウォレットアカウントアドレス。
  - `chainId`, `setChainId`: 現在のチェーンID。
  - `chainName`, `setChainName`: 現在のチェーン名。

- **UIの状態管理**:
  - `disable`, `setDisable`: ボタンの無効化状態。
  - `show`, `setShow`: メインフォームの表示状態。
  - `showNewToken`, `setShowNewToken`: 新しいトークン作成フォームの表示状態。
  - `showDetail`, `setShowDetail`: NFT詳細表示モーダルの表示状態。
  - `isSubmitting`, `setIsSubmitting`: フォーム送信中の状態。

- **NFT関連**:
  - `nfts`, `setNfts`: ユーザーが所有するNFTのリスト。
  - `selectedNft`, `setSelectedNft`: 選択されたNFT。
  - `collections`, `setCollections`: NFTコレクションのリスト。
  - `mintedNfts`, `setMintedNfts`: ミントされたNFTのリスト。
  - `selectedCollection`, `setSelectedCollection`: 選択されたNFTコレクションのID。
  - `selectedCollectionName`, `setSelectedCollectionName`: 選択されたNFTコレクションの名前。
  - `selectedGiftNft`, `setSelectedGiftNft`: 選択されたギフトNFT。

- **その他**:
  - `location`: 現在のURLパスを小文字で取得。

#### モーダルハンドラー

- **`handleClose`**:
  - サイズ情報をリセットし、メインフォームを非表示に。
  - 送信状態を解除。

- **`handleShow`**:
  - メインフォームを表示。

- **`handleCloseNewToken`**, **`handleShowNewToken`**:
  - 新しいトークン作成フォームの表示・非表示を制御。

- **`handleCloseDetail`**, **`handleShowDetail`**:
  - NFT詳細表示モーダルの表示・非表示を制御。
  - `handleShowDetail`では選択されたNFTを設定し、サイズ情報をリセット。

#### サイズ選択ハンドラー

- **`handleRadioChange`**:
  - ラジオボタンで選択されたサイズをステートに設定。
  - 「その他」が選択された場合以外は`otherSize`をリセット。

- **`handleOtherSizeChange`**:
  - 「その他」サイズの入力値をステートに設定。

#### 特定のキーの定義

- **`specificKeys`**:
  - オリジナルTシャツのNFTキーリスト。特定のNFTに対して追加のフォームを表示するために使用。

#### ウォレットの初期化

- **`initializeAccount`**:
  - `getAccount`を呼び出してアカウントを取得。
  - アカウントが存在する場合、`handleAccountChanged`を呼び出してステートを更新。

#### useEffectフック

- **ウォレットイベントのリスナー設定**:
  - `window.ethereum`が存在する場合、アカウント変更やチェーン変更イベントをリスンし、それぞれ`handleAccountChanged`を呼び出す。
  - `window.ethereum`が存在しない場合、`ethereum#initialized`イベントをリスンし、3秒後にウォレットを初期化。

#### レンダリング

- **ナビゲーションバー**:
  - アプリのロゴと「MetaMaskに接続」ボタンを表示。

- **NFT一覧テーブル**:
  - ユーザーが所有するNFTを表形式で表示。
  - 各NFTの画像、コントラクト名、NFT名、もらえるもの、数量、引き換えオプションを表示。
  - 引き換えオプションとしてラジオボタンを提供し、選択すると詳細フォームを表示。

- **詳細フォーム**:
  - 選択されたNFTが特定のキーに含まれる場合（オリジナルTシャツ）、追加のサイズ入力フィールドを表示。
  - それ以外の場合は基本的なユーザー情報フォームを表示。

- **送信ボタン**:
  - `disable`ステートに応じてボタンの有効化・無効化を制御。
  - 送信中はローディングスピナーを表示。

- **フッター**:
  - 「Ideated by Studymeter Inc.」と表示。

---

## まとめ

このReactアプリケーションは、ユーザーが自身のウォレットを通じてNFTを管理し、特定のアクション（例：交換やミント）を実行するための強力なツールを提供しています。主要な機能としてウォレット接続、NFTの取得と表示、詳細操作、新規コレクションのデプロイ、ユーザー情報の入力と送信などがあります。

**改善点と注意点**:

1. **APIキーのセキュリティ**:
   - コード内にハードコードされたAPIキー（AirTableやThirdwebのAPIキー）はセキュリティ上のリスクがあります。環境変数を使用して管理し、コードベースに直接含めないようにしましょう。
   - 例:
     ```javascript
     const AIRTABLE_API_KEY = process.env.REACT_APP_AIRTABLE_API_KEY;
     ```

2. **エラーハンドリングの強化**:
   - 現在のコードでは、一部の非同期処理でエラーハンドリングが不十分です。`try-catch`ブロックを適切に使用し、ユーザーにエラーメッセージを表示するなどの対応が必要です。

3. **コードの再利用性と整理**:
   - フォームのリセットやAPIリクエストなど、共通の機能は関数として切り出し、コードの再利用性と可読性を向上させましょう。

4. **セキュリティの強化**:
   - ユーザー入力に対するバリデーションを実装し、不正なデータの送信を防止します。

5. **UI/UXの向上**:
   - ローディング状態や成功・失敗の通知をより視覚的にわかりやすく表示することで、ユーザー体験を向上させます。

6. **依存関係の管理**:
   - 使用しているライブラリが最新バージョンであることを確認し、セキュリティや機能の向上を図りましょう。

このアプリケーションをさらに発展させるために、上記の改善点に留意しながら機能拡張やユーザー体験の向上を進めてください。具体的な質問や追加の説明が必要な部分があれば、遠慮なくお知らせください。
