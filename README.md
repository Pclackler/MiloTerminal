<p align="center">
  <img src="https://github.com/user-attachments/assets/deff1e50-bf92-46d4-b691-b7ab07ad207c" width="100%" />
</p>

<h1 align="center">Milo Terminal</h1>

<p align="Left">
  C++ · OpenGL · ImGui
  A data-oriented, multithreaded C++ desktop application for tick-level trading strategy backtesting with custom OpenGL charts and a simulated exchange. Built as a companion tool for my live trading system that runs on RHEL. 

  Backtesting tools typically let you test a set of pre-determined indicators or at the most, allow some form of scripting to create your own based on a resolution down to 1m bars.

  With these tools it becomes difficult to simulate and test something like 'at precisely 10:37am last tuesday, look backwards over 3 different time horizons, 5 minutes, 15 minutes, and 30 minutes, and compare the net positive ticks of NVDA vs MSFT'.

  I developed this tool in an attempt to accurately test granular, tick level processing ideas like that, with no limits via a .dll system that helps overcome Windows shared memory implementation limits.
  
</p>



<br>

- **Accurate Playback** Using microsecond timestamps directly from market data ticks synchronized with an internal clock system.
- Loads and processes **50mm+ ticks** with seamless timeline scrubbing and replay at .1–1000x
- **Hot-reloadable DLL strategies** — express any idea in pure C++ without recompiling the host
- **Simulated exchange** with order queue, fills, slippage, and adverse selection modeled from real top-of-book data
- **Zero-copy architecture** — strategy DLLs access host market data directly, no serialization
- **Rapid Deployment** — Shares indentical header files and SoA processing functions as live trading system, greatly minimizes live market implementation. 
- *(coming soon)* Scalable parallel backtesting (1 thread = 1 trading day)
 - *(coming soon)* Minimal compilable example with synthetic data.
---

<details>
<summary><h2>Feature Gallery</h2></summary>


<br>

#### Custom OpenGL Chart Engine

<img src="https://github.com/user-attachments/assets/cbb99fa8-3595-439b-9b23-5520d3c8540a" width="100%"/>

  - Supports data-oriented processing with minimal-copy operations and abstraction bloat 
  - Familiar coordinate system with proper grid scaling, axis scaling
  - Drawing Object system with Smooth splines, rectangles, lines, dynamic shaded areas, transparency, Z-ordering. 
  - Extremely smooth Panning and Zooming / Resizing 
  - OpenGL FBO pipeline keeps processing on GPU, future expandability to WebGL deployment.

 ___ 
#### Visual Replay System
High-performance data-oriented processing with deterministic results validated against live markets. Shown here with 15s, 1m, & 5m lookback windows.

<img src="https://github.com/user-attachments/assets/e07c5a4c-deb1-4f40-9003-17eb6fe47f51"  />

<br><br>
___
#### Playback Speed Dial
Hold **S** and drag to change speed. **Shift+S** snaps to preset increments. Range: .1–1000x.

<p>
  <img src="https://github.com/user-attachments/assets/f9e41401-0d54-4abb-ba83-2ff415b069ee" width="48%" />
  <img src="https://github.com/user-attachments/assets/6bc6b311-284b-4ba4-ad6d-b6ba898951d9" width="48%" />
</p>

<br>
___
#### Transport Controls
Smooth scrubbing across all loaded data and symbols. Start, stop, play, speed controls.

<img src="https://github.com/user-attachments/assets/ec7abc90-792a-441a-9661-66679ca2f440" width="60%" />

<br><br>
___
#### Strategy Hot-Loading
Any novel idea can be expressed in pure C++ and reloaded at runtime — no full recompile, no ~25GB RAM reload. Identical headers and data structures mean a winning strategy can go live **within a single day**.

<img src="https://github.com/user-attachments/assets/036ee060-122d-45d7-9a1e-92b1a61b6c5c" width="60%" />

<br><br>
___
#### Simulated Exchange
Top-of-book depth for queue simulation and matching. Accurate fees and slippage. Supports Market, Limit, StopMarket, OSO Market, and OSO Bracket order types.

<img src="https://github.com/user-attachments/assets/622401aa-9043-42a0-8b69-65ce34215991" width="60%" />

<br><br>
___
#### Builder Pattern API
Fluent builder pattern for strategy configuration. Clean, readable setup with no boilerplate.

<img src="https://github.com/user-attachments/assets/2aeddce8-d60a-4e5d-b487-ead8428b7b00" width="50%" />
<br>
<img src="https://github.com/user-attachments/assets/93fd93f5-a145-42a8-a643-4efe679e28dc" width="50%" />

</details>


---

## Architecture

Data is stored in massive contiguous arrays in the host process. Strategy DLLs have zero-copy access to market data already in memory.

<p align="center">
  
  <img src="https://github.com/user-attachments/assets/752c35db-7a5f-4d4b-93dc-fc88eb6b9cd1" width="80%" />
</p>

Global mutable state is used for speed and simplicity. Global pointers in `.data` resolve to one load instruction. There's no singleton guard check, no factory indirection, no dependency injection framework — just a pointer to the object. The DLL strategy system extends this: `StrategyContext` holds raw pointers to host objects, giving hot-loaded strategies the same direct access path without virtual dispatch.


### Event-Driven vs. Lookback

Much of the tutorial and academic content surrounding algorithmic trading systems is clsoer to event-driven — making a trading decision synchronously on receipt of a trade message, often inline.
My live system has a dedicated process tightly bound to the Linux Kernel to process market data from NIC->Shared memory very quickly. After years of experimentation, I found my current 'lookback' method over shared memory regions to work the best.

<p align="center">
  <img src="https://github.com/user-attachments/assets/6b3a6850-a384-41fd-b713-4f093f762cb7" width="50%" />
</p>

This can be very limiting. My system uses **lookback windows** instead — sampling the last N seconds of ticks at a fixed interval (e.g., 30s of ticks every 25ms). Data-oriented SoA design allows a full lookback scan to complete in under **100μs** under full load during market hours.

<p align="center">
  <img src="https://github.com/user-attachments/assets/d5791d8a-aa55-412d-9cd7-709231c57085" width="60%" />
</p>



## Motivation

I never liked available tools for backtesting trading strategies. Many only operate on 1m bars, are limited in what can be calculated, and most critically — provide inaccurate results. Many researchers backtest ideas that look great in simulation but fail completely on live data.

### Why not config files?

In this context, a 'config' would defeat the purpose of the tool. There are many backtesting tools available that allow 'paramter optimization' of 'known' indicators like RSI,MACD,Volume.. etc. 

This would force all trading logic through a parameter indirection layer. The strategy can only express decisions that the parameter schema anticipated. If the schema doesn't contain a code path for "cancel the order after 13 seconds if \<indicator\> crosses the 70th percentile of today's session", the strategy can't do it. Every parameter not explicitly enumerated is a class of behavior permanently excluded.

This system is designed to create *new* indicators for myself using ticks directly rather than 1m/5m bars, while matching the same processing pipeline as my live trading system.

This greatly reduces the time needed to take a research hypothesis to a live implementation on a physical server that is sending and managing orders. 



---


