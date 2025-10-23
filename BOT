import requests
import time
from web3 import Web3
from web3.middleware import geth_poa_middleware
import json
from base64 import b64decode
from requests.exceptions import RequestException

# Cấu hình
CHAIN_ID = "bsc"  # Thay bằng "97" nếu dùng testnet
MASTER_WALLET = "0xb8293d274e4c45750f6f2da2514de33b5a728bfa".lower()  # Wallet master
BOT_PRIVATE_KEY = "105346693ce74c4956672870baa40250eb3963d86f8b3f91d534372158e22c88"  # Private key ví bot (không có 0x)
WBNB_ADDRESS = "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c"  # WBNB
MIN_TRADE_VALUE = 99  # USD (master trade > $1000)
COPY_TRADE_VALUE = 19  # USD (bot copy $500)
SLIPPAGE = 9.0  # 9%
GAS_PRICE = Web3.to_wei(9, 'gwei')  # 9 Gwei
PRIORITY_FEE = Web3.to_wei(1, 'gwei')  # Priority fee cho Anti-MEV
CHECK_INTERVAL = 10  # Giây
RPC_URL = "https://bsc-dataseed.binance.org/"  # Testnet: https://data-seed-prebsc-1-s1.binance.org:8545/
GMGN_BASE_URL = "https://gmgn.ai/defi/router/v1/bsc"

# PancakeSwap V2 Router
PANCAKE_ROUTER = "0x10ED43C718714eb63d5aA57B78B54704E256024E"  # Testnet: 0xD99D1c33F9fC3444f8101754aBC46c52416550D1
PANCAKE_ABI = [
    {
        "inputs": [
            {"internalType": "uint256", "name": "amountOutMin", "type": "uint256"},
            {"internalType": "address[]", "name": "path", "type": "address[]"},
            {"internalType": "address", "name": "to", "type": "address"},
            {"internalType": "uint256", "name": "deadline", "type": "uint256"}
        ],
        "name": "swapExactETHForTokens",
        "outputs": [{"internalType": "uint256[]", "name": "amounts", "type": "uint256[]"}],
        "stateMutability": "payable",
        "type": "function"
    }
]

# ERC20 ABI cho approve
ERC20_ABI = [
    {
        "inputs": [
            {"internalType": "address", "name": "spender", "type": "address"},
            {"internalType": "uint256", "name": "amount", "type": "uint256"}
        ],
        "name": "approve",
        "outputs": [{"internalType": "bool", "name": "", "type": "bool"}],
        "stateMutability": "nonpayable",
        "type": "function"
    }
]

# Kết nối Web3
w3 = Web3(Web3.HTTPProvider(RPC_URL))
w3.middleware_onion.inject(geth_poa_middleware, layer=0)
if not w3.is_connected():
    raise Exception("Không kết nối được với BSC node")
bot_account = w3.eth.account.from_key(BOT_PRIVATE_KEY)
bot_address = bot_account.address
pancake_contract = w3.eth.contract(address=PANCAKE_ROUTER, abi=PANCAKE_ABI)

def retry_request(func):
    """Retry API request tối đa 3 lần"""
    for _ in range(3):
        try:
            return func()
        except RequestException as e:
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Retry lỗi: {e}")
            time.sleep(1)
    return None

def get_dexscreener_trades(wallet):
    """Lấy recent trades từ DEXScreener cho wallet"""
    def fetch():
        url = f"https://api.dexscreener.com/latest/dex/trades/{CHAIN_ID}?wallet={wallet}"
        resp = requests.get(url, timeout=5)
        data = resp.json()
        trades = []
        for pair in data.get('pairs', []):
            for trade in pair.get('trades', []):
                trade['pairAddress'] = pair['pairAddress']
                trades.append(trade)
        return trades
    return retry_request(fetch) or []

def get_gmgn_token_price(token_address):
    """Lấy giá USD của token từ GMGN API (simulate swap WBNB -> Token)"""
    amount_in = w3.to_wei(0.01, 'ether')  # 0.01 WBNB
    def fetch():
        url = f"{GMGN_BASE_URL}/tx/simulate_route_exact_in"
        params = {
            'token_in_address': WBNB_ADDRESS,
            'token_out_address': token_address,
            'in_amount': str(amount_in),
            'from_address': bot_address,
            'slippage': SLIPPAGE
        }
        resp = requests.get(url, params=params, timeout=5)
        data = resp.json()
        if data.get('success') and data['data'].get('routes'):
            route = data['data']['routes'][0]
            amount_out = int(route.get('amount_out', 0))
            amount_out_usd = float(route.get('amount_out_usd', 0))
            if amount_out > 0:
                price_usd = amount_out_usd / (amount_out / 10**18)  # Giả sử 18 decimals
                return price_usd
        return None
    return retry_request(fetch)

def get_gmgn_swap_route(token_address, amount_in_wei):
    """Lấy swap route từ GMGN API"""
    def fetch():
        url = f"{GMGN_BASE_URL}/tx/get_swap_route"
        params = {
            'token_in_address': WBNB_ADDRESS,
            'token_out_address': token_address,
            'in_amount': str(amount_in_wei),
            'from_address': bot_address,
            'slippage': SLIPPAGE
        }
        resp = requests.get(url, params=params, timeout=5)
        data = resp.json()
        if data.get('success'):
            return data['data'].get('raw_tx', {}).get('swapTransaction')
        return None
    return retry_request(fetch)

def approve_token(token_address):
    """Tự động approve token cho PancakeSwap Router"""
    try:
        token_contract = w3.eth.contract(address=token_address, abi=ERC20_ABI)
        allowance = token_contract.functions.allowance(bot_address, PANCAKE_ROUTER).call()
        if allowance < w3.to_wei(1_000_000, 'ether'):  # Nếu allowance thấp
            tx = token_contract.functions.approve(
                PANCAKE_ROUTER,
                2**256 - 1  # MAX_INT
            ).build_transaction({
                'from': bot_address,
                'gas': 100_000,
                'gasPrice': GAS_PRICE,
                'maxPriorityFeePerGas': PRIORITY_FEE,
                'nonce': w3.eth.get_transaction_count(bot_address)
            })
            signed_tx = w3.eth.account.sign_transaction(tx, BOT_PRIVATE_KEY)
            tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
            w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Approved token {token_address} cho PancakeSwap")
            return True
        return True
    except Exception as e:
        print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Lỗi approve token: {e}")
        return False

def execute_swap(base64_tx):
    """Thực hiện swap từ GMGN route"""
    try:
        raw_tx = b64decode(base64_tx)
        tx = {
            'data': raw_tx,
            'gasPrice': GAS_PRICE,
            'maxPriorityFeePerGas': PRIORITY_FEE,  # Anti-MEV
            'from': bot_address,
            'nonce': w3.eth.get_transaction_count(bot_address)
        }
        signed_tx = w3.eth.account.sign_transaction(tx, BOT_PRIVATE_KEY)
        tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
        print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Copy trade $19 executed. Tx: {tx_hash.hex()}")
        return tx_hash
    except Exception as e:
        print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Lỗi execute swap: {e}")
        return None

def parse_master_trade(trade):
    """Phân tích trade từ DEXScreener"""
    try:
        if trade['buyer'].lower() == MASTER_WALLET and trade['side'] == 'buy':
            amount_token = float(trade['amount'])  # Amount token mua
            token_address = trade['token0']['address'] if trade['token1']['address'].lower() == WBNB_ADDRESS.lower() else trade['token1']['address']
            pair_address = trade['pairAddress']
            return amount_token, token_address, pair_address
        return None, None, None
    except:
        return None, None, None

def get_liquidity(pair_address):
    """Kiểm tra liquidity từ DEXScreener"""
    def fetch():
        url = f"https://api.dexscreener.com/latest/dex/pairs/{CHAIN_ID}/{pair_address}"
        resp = requests.get(url, timeout=5)
        data = resp.json()
        return float(data.get('pair', {}).get('liquidity', {}).get('usd', 0))
    return retry_request(fetch) or 0

def get_wnb_price():
    """Lấy giá WBNB USD từ Coingecko"""
    def fetch():
        resp = requests.get("https://api.coingecko.com/api/v3/simple/price?ids=binancecoin&vs_currencies=usd", timeout=5)
        return resp.json()['binancecoin']['usd']
    return retry_request(fetch) or 600  # Fallback $600

def main():
    print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Bot tự động copy trade DEXScreener + GMGN (Dynamic Token/WBNB, $19, Auto-MEV, Auto Approval) đang chạy...")
    last_time = None
    
    while True:
        try:
            trades = get_dexscreener_trades(MASTER_WALLET)
            for trade in trades[-5:]:  # 5 trades mới nhất
                t_time = trade.get('timestamp', 0)
                if last_time is None or t_time > last_time:
                    amount_token, token_address, pair_address = parse_master_trade(trade)
                    if amount_token and token_address:
                        liquidity_usd = get_liquidity(pair_address)
                        if liquidity_usd < 10000:
                            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Ignore token {token_address}: Liquidity ${liquidity_usd:.2f} < $10000")
                            continue
                        price_usd = get_gmgn_token_price(token_address)
                        if price_usd:
                            value_usd = amount_token * price_usd
                            if value_usd > MIN_TRADE_VALUE:
                                print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Phát hiện buy master: {amount_token:.4f} Token ({token_address}) at ${price_usd:.4f}. Value: ${value_usd:.2f}")
                                # Auto approve token
                                if not approve_token(token_address):
                                    continue
                                # Copy $19 bằng WBNB
                                wbnb_price = get_wnb_price()
                                wbnb_needed = COPY_TRADE_VALUE / wbnb_price
                                amount_in_wei = w3.to_wei(wnb_needed, 'ether')
                                base64_tx = get_gmgn_swap_route(token_address, amount_in_wei)
                                if base64_tx:
                                    execute_swap(base64_tx)
                                last_time = t_time
                            else:
                                print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Ignore trade nhỏ: ${value_usd:.2f}")
                        else:
                            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Không lấy được giá cho token {token_address}")
            time.sleep(CHECK_INTERVAL)
        except Exception as e:
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Lỗi: {e}")
            time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    main()
