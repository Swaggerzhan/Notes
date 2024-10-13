# Paxos

## basic-paxos


<table>
    <tr>
        <th>Phase/Role</th><th>Proposer</th><th>Acceptor</th>
    </tr>
    <tr>
        <td rowspan="2">Phase1(Prepare)</td>
        <td>
            生成唯一且递增的N，这里记为Proposer(N)，江请求发送给Acceptor
        </td>
        <td style="white-space: nowrap;" rowspan="2">
            Acceptor收到来自Proposer的请求，根据本地已有内容作出判断：<br>
            case1 Proposer(N) <= Acceptor(N):<br>
            &nbsp;&nbsp;&nbsp;&nbsp; 拒绝投票，返回。<br>
            case2 Proposer(N) > Acceptor(N):<br>
            &nbsp;&nbsp;&nbsp;&nbsp;投赞成票，Acceptor记住这个新的N，然后返回前把自己<br>
            &nbsp;&nbsp;&nbsp;&nbsp;已经接受的提案(这里记做N1和V1)也一并返回，如果没有，<br>&nbsp;&nbsp;&nbsp;&nbsp;
            则返回NULL。
        </td>
    </tr>
    <tr>
        <td style="white-space: nowrap;">
        Proposer根据Acceptor的返回情况再作出判断：<br>
        case1 收到的票数没有超过总数的一半：<br>
        &nbsp;&nbsp;&nbsp;&nbsp;结束，本次提案没有通过。<br>
        case2 收到的票数超过半数：<br>
        &nbsp;&nbsp;&nbsp;&nbsp;进入第二阶段
        </td>
    </tr>
    <tr>
        <td rowspan="2">Phase2(Accept)</td>
        <td>
        Proposer从一阶段所有Acceptor中，选择他们返回的结果中N1最大的提案，将这个N1的提案内容V1，作为本次的提案，如果都是NULL，则Proposer可以自己决定V，然后向Acceptor发起Accept(N, V)请求。
        </td>
        <td rowspan="2">
        Acceptor对比本地情况，决定是否赞成投票：<br>
        case1 Proposer(N) < Acceptor(N)：<br>&nbsp;&nbsp;&nbsp;&nbsp;
        投反对票，返回当前N。<br>
        case2 Proposer(N) >= Acceptor(N)：<br>&nbsp;&nbsp;&nbsp;&nbsp;
        投赞成票，接受这个提案，记住N和V
        </td>
    </tr>
    <tr>
        <td>
        Proposer根据Accept二阶段返回情况判断：<br>
        case1 票数没有超过半数，提案N没有通过。<br>
        case2 票数超半，则提案(N, V)提案达成，结束。
        </td>
    </tr>
</table>

basic-paxos的本质其实是一种分布式占坑：
1. Prepare流程保证Proposer能够占到半数的坑，然后才会发起Acceptor请求。
2. Accept流程保证一定能否发现之前有人“成功”占过坑，并且能替他们完成“完整的占坑”的请求。
