## Backtrader gym environment
The idea is to implement  OpenAI Gym environment for Backtrader backtesting/trading library to apply
reinforcement learning algorithms in algo-trading domain.

Backtrader is open-source algorithmic trading library, well structured and maintained at:
http://github.com/mementum/backtrader
http://www.backtrader.com/

OpenAI Gym is...,
well, everyone knows Gym.
http://github.com/openai/gym

### Update 9.06.17: Basic work done. Few days to first working alpha.


####Outline:

Consider reinforcement learning setup for equity/currency trading:
- agent action space either discrete ('buy', 'sell', 'close'[position], 'hold'[do nothing])
or continuous (portfolio reballancing vector, as for portfolio optimisation setting);
- environment is episodic: maximum  episode duration and episode termination conditions
  are set;
- for every timestep of the episode agent is given environment state observation as tensor of last
  m price open/high/low/close values for every equity considered and based on that information is making
  trading/investemeent decisions.
- agent's goal is to maximize cumulative capital;
- 'market liquidity' and 'capital impact' assumptions are met.

####Data selection options for backtest agent training:

- random sampling:
  historic price dchange dataset is divided to training, cross-validation and testing subsets.
  Since agent actions do not influence market,it is posible to randomly sample continuous subset
  of training data for every episode. This is most data-efficient method.
  Cross-validation and testing performed later as usual on most "recent" data;
- sequential sampling:
  full dataset is feeded sequentially as if agent is performing real-time trading,
  episode by episode. Most reality-like, least data-efficient;
- sliding time-window sampling:
  mixture of above, episde is sampled randomly from comparatively short time period, sliding from
  furthest to most recent training data.
  NOTE: only random sampling is currently implemented.

####Environment engine:

  BTgym uses Backtrader framework for actual environment computations, see:
https://www.backtrader.com/docu/index.html for extensive documentation.
In brief, assuming basic backtrader and OpenAI Gym knowledge:
- User defines backtrading engine parameters by composing Backtrader.Cerebro() subclass,
  data feed as subclass of Backtrader.datafeeds() and passes it as arguments when making BTgym
  environment. See backtrader docs for details.
- Envoronment starts separate server process responsible for rendering gym environment
  queries like env.reset() and env.step() by repeatedly sampling episodes form given data and running
  Cerebro enginge on it. See OpenAI Gym for details.

See notebooks in examples directory.




```
Schematic data flow:

            BacktraderEnv                                  RL alorithm
                                           +-+
   (episode mode)  +<-----<action>--- -----| |<--------------------------------------+
          |        |                       |e|                                       |
          +<------>+------<state observ.>->|n|--->[feature  ]---><state>--+->[agent]-+
          |        |      <      matrix >  |v|    [estimator]             |     |
          |        |                       |.|                            |     |
    [Backtrader]   +------<portfolio  >--->|s|--->[reward   ]---><reward>-+     |
    [Server    ]   |      <statistics>     |t|    [estimator]                   |
       |           |                       |e|                                  |
       |           +------<is_done>------->|p|--+>[runner]<-------------------->+
  (control mode)   |                       | |  |    |
       |           +------<aux.info>--- -->| |--+    |
       |                                   +-+       |
       +--<'_stop'><------------------->|env._stop|--+
       |                                             |
       +--<'_reset'><------------------>|env.reset|--+

```

TODO: REWRITE
 Notes:
 1. While feature estimator and 'MDP state composer' are traditionally parts of RL algorithms,
    reward estimation is often performed inside environment. In case of portfolio optimisation
    reward function can be tricky, so it is reasonable to make it easyly accessable inside RL algorithm,
    computed by [reward estimator] module and based on some set of portfolio statisics.
 2. [state matrix], returned by Environment is 2d [m,n] array of floats, where m - number of Backtrader
    Datafeed values: v[-n], v[-n+1], v[-n+2],...,v[0] i.e. from present step to n steps back, and
    every v[i] is itself a vector of m features (open, close,...,volume,..., mov.avg., etc.).
    - in case of n=1 process is obviously POMDP. Ensure MDP property by 'frame stacking' or/and
      employing reccurent function approximators. When n>>1 process [somehow] approaches MDP (by means of
      Takens' delay embedding theorem).
    - features are defined by BTgymStrategy,wich is added. TODO: rewrite
    - same holds for portfolio statistics.
    <<TODO: pass features and stats as parameters of environment>> DONE,
 3. Action space is discrete with basic actions: 'buy', 'sell', 'hold', 'close',
    and control action 'done' for early epidsode termination.
    <<!:very incomplete: order amounts? ordering logic not defined>>
 4. This environment is meant to be [not nessesserily] paired with Tensorforce RL library,
    that's where [runner] module came from.

 5. Why Gym, not Universe VNC environment?
    For algorithmic trading, clearly, vnc-type environment fits much better.
    But to the best of my knowledge, OpenAI yet to publish docs on custom Universe VNC environment creation.

 6. Why Backtrader library, not Zipline/PyAlgotrader etc.?
    Those are excellent platforms, but what I really like about Backtrader is open programming logic
    and ease of customisation. You dont't need to do tricks, say, to disable automatic calendar fetching
    as with Zipline. I mean, it's nice feature and very convinient for trading people but prevents from
    correctly running intraday trading strategies. IMO Backtrader is simply better suited for this kind of experiments.

 7. Why Forex data?
    Obviously environment is data/market agnostic. Backtesting dataset size is what matters.
    Deep Q-value algorithm, most sample efficient among deep RL, take 1M steps just to lift off.
    1 year 1 minute FX data contains about 300K samples.Feeding several years of data makes it realistic
    to expect algorithm to converge for intraday trading (~1000-1500 steps per episode).
    That's just preliminary experiment assumption, not proved!

####Server inner operation details:
    Backtrader server starts when BacktraderEnv is instantiated, runs as separate process, follows
Request/Reply pattern (every request should be paired with reply message) and operates one of two modes:
1. Control mode: initial mode, accepts only '_reset', '_stop' and '_getstat' messages. Any other message is ignored
   and replied with simple info messge. Shuts down upon recieving 'stop' via environment close() method,
   goes to episode mode upon 'reset' (via env.reset()) and send last run episode statistic (if any) upon '_getstat'
2. Episode mode: runs episode following passed bt.Cerebro subclass logic and parameters. Accepts <action> messages,
   returns tuple <[state observ. tensor], [reward], [is_done], [aux.info]>.
   Finishes episode upon recieving <action>='_done' or according to Cerebro logic, falls
   back to control mode.
   Before every episode runtime, BTserver samples episode data and adds it to bt.Cerebro instance
along with specific _BTgymAnalyzer.The service of this hidden Analyzer is twofold:
first, enables strategy-environment communication by calling RL-related BTgymStrategy methods:
get_state(), get_reward(), get_info() and is_done() [- see below];
second, controls episode termination by specified environment conditions.
   Runtime (server 'Episode mode'): after preparing environment initial state by running start(), prenext() methods
for BTgymStrategy, server halts and waits for incoming agent action. Upon recieving action, server performs all
nesessery Strategy next() actions (e.g. issues orders, computes observations etc.),
composes environment response and sends it back to environment wrapper.
Repeats until episode termination conditions are met.

####Simple workflow:
1. Define strategy by subclassing BTgymStrategy, which is by itself is subclass of bt.Strategy and
   controls Environment inner dynamics and backtesting logic. As for RL-specific part,any State,
   Reward and Info computation logic can be implemented by overriding get_state(), get_reward(),
   get_info(), is_done() and set_datalines() methods.
   One can always go deeper and override __init__ () and next() methods for desired
   server Cerebro engine behaviour, including order execution logic etc.

2. Instantiate Cerbro, add startegy and backtrader Analyzers an Observers (if needed).

3. Define dataset by passing csv datafile and parameters to BTgymData instance.
    BTgymData() is simply Backtrader.feeds class wrapper, which pipes
    CSV[source]-->pandas[for efficient sampling]-->bt.feeds routine
    and implements random episode data sampling.
    Suggested usage:
        ---user defined ---
        D = BTgymData(<filename>,<params>)
        ---inner BTgymServer routine---
        D.read_csv(<filename>)
        Repeat until bored:
            Episode = D.get_sample()
            DataFeed = Episode.to_btfeed()
            C = bt.Cerebro()
            C.adddata(DataFeed)
            C.run()

4. Instantiate (or make) gym environment by passing Cerebro and BTgymData instance along with other kwargs

5. Run your favorite RL algorithm.

See notebooks in examples directory.




