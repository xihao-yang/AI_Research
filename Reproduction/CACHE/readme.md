###

# 1. Goal

* Reproduce CACHE paper pipeline on server
* Current status: preprocessing + AST mining + vocab + small-scale training run completed

# 2. Environment

* Ubuntu
* Python 3.10.12
* Java 11 actually used for current run
* Defects4J isolated path

# 3. What was fixed

* patch file suffix/path parsing
* bears_mapping path issue
* Defects4J-only filtering / patch loading adjustments
* dataset_reader_binary.py fixed/buggy pairing logic
* dataset_builder.py location tensor issue
* training stability tuning on CPU server

# 4. Current result

* dataset_small_try/java generated successfully
* training reached epoch 0 and printed metrics
* full 40-epoch run not finished yet（TBD）

# 5. Known issues

* server CPU steal time is high
* metrics on tiny dataset are not meaningful yet
* warnings about dropout and undefined precision/recall/f1 on small split

* There is some potential problems about results_v1.json
