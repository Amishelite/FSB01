import { Connection, Keypair, PublicKey, Transaction, ComputeBudgetProgram } from '@solana/web3.js';
import { KaminoFlashLoan } from '@kamino-finance/flash-loan';
import { OrcaPool, OrcaFactory } from '@orca-so/sdk';
import { RaydiumPoolManager } from '@raydium-io/raydium-sdk';
import { Bundle } from '@jito/bundler';
import BN from 'bn.js';

// 1. CONFIG
const RPC_URL = 'https://jito.xyz/rpc';
const WALLET = Keypair.fromSecretKey(/* Your PK */);
const FLASH_LOAN_PROGRAM = new PublicKey('KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD');

// 2. SETUP
const connection = new Connection(RPC_URL, {
  commitment: 'confirmed',
  wsEndpoint: 'wss://jito.xyz/ws'
});
const orca = new OrcaFactory(connection);
const raydium = new RaydiumPoolManager(connection);
const kamino = new KaminoFlashLoan(connection, FLASH_LOAN_PROGRAM);
const jitoBundle = new Bundle(connection, WALLET);

// 3. NEW POOL DETECTOR
class NewPoolScanner {
  private seenPools = new Set<string>();
  private recentPools: {pubkey: PublicKey, timestamp: number}[] = [];

  async scan(intervalMs = 15000) {
    console.log('Scanning for new pools...');
    
    // Get latest pools from both DEXs
    const [orcaPools, raydiumPools] = await Promise.all([
      orca.getPools(),
      raydium.getPools()
    ]);

    // Filter new pools (<5 mins old)
    const now = Date.now();
    const newPools = [...orcaPools, ...raydiumPools].filter(pool => {
      const poolKey = pool.poolAddress.toBase58();
      const isNew = !this.seenPools.has(poolKey);
      if (isNew) {
        this.seenPools.add(poolKey);
        this.recentPools.push({
          pubkey: pool.poolAddress,
          timestamp: now
        });
        return true;
      }
      return false;
    });

    // Cleanup old entries
    this.recentPools = this.recentPools.filter(
      p => now - p.timestamp < 300_000 // 5 mins
    );

    return newPools;
  }

  getFreshPools() {
    const fiveMinutesAgo = Date.now() - 300_000;
    return this.recentPools
      .filter(p => p.timestamp >= fiveMinutesAgo)
      .map(p => p.pubkey);
  }
}

// 4. IMBALANCE ANALYZER
async function analyzePool(pool: OrcaPool | RaydiumPool) {
  const data = pool instanceof OrcaPool 
    ? await pool.getData()
    : await pool.getPoolData();

  // Skip unstable pools
  if (data.tvl < 1000 || data.volume24h < 5000) return null;

  const feeRatio = 'feeA' in data 
    ? data.feeA.toNumber() / (data.feeB.toNumber() || 1)
    : data.feeTokenA / (data.feeTokenB || 1);

  return {
    pool,
    dex: pool instanceof OrcaPool ? 'Orca' : 'Raydium',
    imbalance: Math.abs(1 - feeRatio),
    timestamp: Date.now()
  };
}

// 5. FLASH LOAN SKIM EXECUTOR
async function executeSkim(target: Awaited<ReturnType<typeof analyzePool>>) {
  if (!target) return false;

  try {
    // Build flash loan strategy
    const poolData = await target.pool.getData();
    const loanAmount = new BN(Math.min(
      poolData.reserveA.toNumber() * 0.03,
      poolData.reserveB.toNumber() * 0.03
    ));

    const swapTx = await target.pool.swap({
      amount: loanAmount,
      inputToken: target.imbalance > 1 ? poolData.tokenA : poolData.tokenB,
      outputToken: target.imbalance > 1 ? poolData.tokenB : poolData.tokenA,
      slippage: 0.3,
      wallet: WALLET.publicKey
    });

    // MEV Protection
    swapTx.transaction.add(
      ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 250_000 }),
      ComputeBudgetProgram.setComputeUnitLimit({ units: 300_000 })
    );

    // Execute with Jito bundle
    const { txSignature } = await kamino.execute(
      new FlashLoanStrategy({
        loanAmount,
        loanToken: target.imbalance > 1 ? poolData.tokenA : poolData.tokenB,
        actions: [swapTx]
      }),
      WALLET
    );

    console.log(`Skimmed new ${target.dex} pool: ${txSignature}`);
    return true;
  } catch (err) {
    console.error(`Failed on ${target.pool.poolAddress}: ${err.message}`);
    return false;
  }
}

// 6. MAIN BOT LOOP
async function run() {
  const scanner = new NewPoolScanner();
  
  // Initial scan
  await scanner.scan();

  setInterval(async () => {
    try {
      // Detect new pools
      await scanner.scan();
      const freshPools = scanner.getFreshPools();
      
      // Analyze and skim
      for (const poolPubkey of freshPools) {
        const pool = await OrcaPool.fetch(connection, poolPubkey)
          .catch(() => RaydiumPool.fetch(connection, poolPubkey));
        
        const analysis = await analyzePool(pool);
        if (analysis?.imbalance > 0.1) { // 10%+ imbalance
          await executeSkim(analysis);
        }
      }
    } catch (err) {
      console.error('Bot error:', err);
    }
  }, 15000); // Scan every 15s
}

// START
run().catch(console.error);
