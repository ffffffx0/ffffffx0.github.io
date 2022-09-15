---
layout: post
title: "raft/hashicorp raft实现源码笔记"
keywords: ["raft"]
description: "raft"
category: "raft"
tags: ["go","raft"]
---

apply
```
case LogCommand:
  start := time.Now()
  resp = r.fsm.Apply(req.log)
  metrics.MeasureSince([]string{"raft", "fsm", "apply"}, start)
  //
func (c *FSM) Apply(log *raft.Log) interface{} {
	buf := log.Data
	msgType := structs.MessageType(buf[0])

	// Check if this message type should be ignored when unknown. This is
	// used so that new commands can be added with developer control if older
	// versions can safely ignore the command, or if they should crash.
	ignoreUnknown := false
	if msgType&structs.IgnoreUnknownTypeFlag == structs.IgnoreUnknownTypeFlag {
		msgType &= ^structs.IgnoreUnknownTypeFlag
		ignoreUnknown = true
	}

	// Apply based on the dispatch table, if possible. 
	if fn := c.apply[msgType]; fn != nil {
		return fn(buf[1:], log.Index)
	}

	// Otherwise, see if it's safe to ignore. If not, we have to panic so
	// that we crash and our state doesn't diverge.
	if ignoreUnknown {
		c.logger.Warn("ignoring unknown message type, upgrade to newer version", "type", msgType)
		return nil
	}
	panic(fmt.Errorf("failed to apply request: %#v", buf))
}
```
这里注意下fn的的注册registerCommand
```
func init() {
	registerCommand(structs.RegisterRequestType, (*FSM).applyRegister)
	registerCommand(structs.DeregisterRequestType, (*FSM).applyDeregister)
	registerCommand(structs.KVSRequestType, (*FSM).applyKVSOperation)
	registerCommand(structs.SessionRequestType, (*FSM).applySessionOperation)
	// DEPRECATED (ACL-Legacy-Compat) - Only needed for v1 ACL compat
	registerCommand(structs.ACLRequestType, (*FSM).applyACLOperation)
	registerCommand(structs.TombstoneRequestType, (*FSM).applyTombstoneOperation)
	registerCommand(structs.CoordinateBatchUpdateType, (*FSM).applyCoordinateBatchUpdate)
	registerCommand(structs.PreparedQueryRequestType, (*FSM).applyPreparedQueryOperation)
	registerCommand(structs.TxnRequestType, (*FSM).applyTxn)
	registerCommand(structs.AutopilotRequestType, (*FSM).applyAutopilotUpdate)
	registerCommand(structs.IntentionRequestType, (*FSM).applyIntentionOperation)
	registerCommand(structs.ConnectCARequestType, (*FSM).applyConnectCAOperation)
	registerCommand(structs.ACLTokenSetRequestType, (*FSM).applyACLTokenSetOperation)
	registerCommand(structs.ACLTokenDeleteRequestType, (*FSM).applyACLTokenDeleteOperation)
	registerCommand(structs.ACLBootstrapRequestType, (*FSM).applyACLTokenBootstrap)
	registerCommand(structs.ACLPolicySetRequestType, (*FSM).applyACLPolicySetOperation)
	registerCommand(structs.ACLPolicyDeleteRequestType, (*FSM).applyACLPolicyDeleteOperation)
	registerCommand(structs.ConnectCALeafRequestType, (*FSM).applyConnectCALeafOperation)
	registerCommand(structs.ConfigEntryRequestType, (*FSM).applyConfigEntryOperation)
	registerCommand(structs.ACLRoleSetRequestType, (*FSM).applyACLRoleSetOperation)
	registerCommand(structs.ACLRoleDeleteRequestType, (*FSM).applyACLRoleDeleteOperation)
	registerCommand(structs.ACLBindingRuleSetRequestType, (*FSM).applyACLBindingRuleSetOperation)
	registerCommand(structs.ACLBindingRuleDeleteRequestType, (*FSM).applyACLBindingRuleDeleteOperation)
	registerCommand(structs.ACLAuthMethodSetRequestType, (*FSM).applyACLAuthMethodSetOperation)
	registerCommand(structs.ACLAuthMethodDeleteRequestType, (*FSM).applyACLAuthMethodDeleteOperation)
	registerCommand(structs.FederationStateRequestType, (*FSM).applyFederationStateOperation)
	registerCommand(structs.SystemMetadataRequestType, (*FSM).applySystemMetadataOperation)
}
```
