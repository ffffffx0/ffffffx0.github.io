---
title: Flink Exactly-Once 
tagline: ""
category : flink
layout: post
tags : [flink, streamsets, realtime]
---


source 端

checkpoint后 kafka consumer提交offset, FlinkKafkaConsumerBase.notifyCheckpointComplete-->fetcher.commitInternalOffsetsToKafka--> AbstractFetcher.doCommitInternalOffsetsToKafka
```
    public final void notifyCheckpointComplete(long checkpointId) throws Exception {
        if (!running) {
            LOG.debug("notifyCheckpointComplete() called on closed source");
            return;
        }

        final AbstractFetcher<?, ?> fetcher = this.kafkaFetcher;
        if (fetcher == null) {
            LOG.debug("notifyCheckpointComplete() called on uninitialized source");
            return;
        }

        if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
            // only one commit operation must be in progress
            if (LOG.isDebugEnabled()) {
                LOG.debug(
                        "Consumer subtask {} committing offsets to Kafka/ZooKeeper for checkpoint {}.",
                        getRuntimeContext().getIndexOfThisSubtask(),
                        checkpointId);
            }

            try {
                final int posInMap = pendingOffsetsToCommit.indexOf(checkpointId);
                if (posInMap == -1) {
                    LOG.warn(
                            "Consumer subtask {} received confirmation for unknown checkpoint id {}",
                            getRuntimeContext().getIndexOfThisSubtask(),
                            checkpointId);
                    return;
                }

                @SuppressWarnings("unchecked")
                Map<KafkaTopicPartition, Long> offsets =
                        (Map<KafkaTopicPartition, Long>) pendingOffsetsToCommit.remove(posInMap);

                // remove older checkpoints in map
                for (int i = 0; i < posInMap; i++) {
                    pendingOffsetsToCommit.remove(0);
                }

                if (offsets == null || offsets.size() == 0) {
                    LOG.debug(
                            "Consumer subtask {} has empty checkpoint state.",
                            getRuntimeContext().getIndexOfThisSubtask());
                    return;
                }

                fetcher.commitInternalOffsetsToKafka(offsets, offsetCommitCallback);
            } catch (Exception e) {
                if (running) {
                    throw e;
                }
                // else ignore exception if we are no longer running
            }
        }
    }
```
