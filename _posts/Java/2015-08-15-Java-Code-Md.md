---
title: Kafka Schedule and Rmq NettyEncoder
tagline: ""
category : kafka
layout: post
tags : [kafka, rmq, scala, java]
---
### JAVA CODE：
```
public class NettyEncoder extends MessageToByteEncoder<RemotingCommand> {
    private static final Logger log = LoggerFactory.getLogger(RemotingHelper.RemotingLogName);

    @Override
    public void encode(ChannelHandlerContext ctx, RemotingCommand remotingCommand, ByteBuf out)
            throws Exception {
        try {
            ByteBuffer header = remotingCommand.encodeHeader();
            out.writeBytes(header);
            byte[] body = remotingCommand.getBody();
            if (body != null) {
                out.writeBytes(body);
            }
        } catch (Exception e) {
            log.error("encode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            if (remotingCommand != null) {
                log.error(remotingCommand.toString());
            }
            RemotingUtil.closeChannel(ctx.channel());
        }
    }
}
```

### SCALA CODE：

```
  def schedule(name: String, fun: ()=>Unit, delay: Long, period: Long, unit: TimeUnit) = {
    debug("Scheduling task %s with initial delay %d ms and period %d ms."
        .format(name, TimeUnit.MILLISECONDS.convert(delay, unit), TimeUnit.MILLISECONDS.convert(period, unit)))
    ensureStarted
    val runnable = Utils.runnable {
      try {
        trace("Begining execution of scheduled task '%s'.".format(name))
        fun()
      } catch {
        case t: Throwable => error("Uncaught exception in scheduled task '" + name +"'", t)
      } finally {
        trace("Completed execution of scheduled task '%s'.".format(name))
      }
    }
    if(period >= 0)
      executor.scheduleAtFixedRate(runnable, delay, period, unit)
    else
      executor.schedule(runnable, delay, unit)
  }
```
