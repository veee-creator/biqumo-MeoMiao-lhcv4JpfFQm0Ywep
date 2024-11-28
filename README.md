
#### 前言


常说c\#、java是面向对象的语言，但我们平时都是在用面向过程的思维写代码，实现业务逻辑像记流水账一样，大篇if else的判断；对业务没有抽象提炼、代码没有分层。随着需求变化、功能逐步拓展、业务逻辑逐渐复杂；代码越来越长、if else嵌套越来越多，代码会变成程序员都厌恶的"屎山"。这种代码后期维护成本非常高、牵一发而动全身、改一处逻辑战战兢兢。假如我们完成开发任务交差，后期维护不关自己的事；但是长期做重复的CRUD、记流水账对我们没有好处。虽然项目不是自己的，但是时间是自己的，这样几年过去似乎没有精进变化，长期下去年龄增大会逐渐丧失竞争力。


下面记录今天开发的一个小功能，演示一步一步重构的过程。


#### 需求


1. 有一个智能识别的api给用户调用。角色有两个：管理员、普通用户。管理员不限次数调用，普通用户每日限用五次。


简单实现，只判断如果是普通用户就检查次数，不满足就返回提示：



```
if (service.isNormalUser() && service.freeNumUseUp()) {
    return AjaxResult.error("普通用户免费识别次数已使用完！");
}

// todo:调用识别接口

```

2. 功能演变：普通用户可充值后升级为VIP用户，VIP用户在有效期内不限次数使用，过期以后降为普通用户。
增加VIP角色的检查后：



```
if (service.isVipUser() && service.vipUserExpire()) {
    return AjaxResult.error("会员已到期！");
}
if (service.isNormalUser() && service.freeNumUseUp()) {
    return AjaxResult.error("普通用户免费识别次数已使用完！");
}

// todo:调用识别接口

```

以上修改的问题：普通用户充值以后，是增加一个VIP的角色而不是把原普通用户角色更新为VIP角色。此时这个用户有两个角色，那么上面的代码先判断VIP角色是否到期是没问题的，但是下面又判断了是否为普通用户就有问题了，因为他有两个角色呀，VIP未到期时第2个条件也满足了会给出不合理的提示。怎么改，首先想到的是不是检查VIP后就不检查普通用户了？于是修改为：



```
if (service.isVipUser()) {
    if (service.vipUserExpire()) {
        return AjaxResult.error("会员已到期！");
    }
} else {
    if (service.isNormalUser() && service.freeNumUseUp()) {
        return AjaxResult.error("普通用户免费识别次数已使用完！");
    }
}

// todo:调用识别接口

```

以上仍然有问题，如果是VIP角色就不会检查普通用户角色了，可是按需求VIP到期以后他还具有普通用户角色，可以在每天免费次数内使用。于是再改：



```
boolean dontPass = service.isVipUser() && service.vipUserExpire();
if (dontPass) {
    dontPass = service.isNormalUser() && service.freeNumUseUp();
    if (dontPass) {
        return AjaxResult.error("普通用户免费识别次数已使用完！");
    } else {
        return AjaxResult.error("会员已到期！");
    }
}

// todo:调用识别接口

```

以上修改可以满足VIP和普通用户的检查了，还差了管理员的判断，还要再嵌套：



```
boolean dontPass = !service.isAdmin();
if (dontPass) {
    dontPass = service.isVipUser() && service.vipUserExpire();
    if (dontPass) {
        dontPass = service.isNormalUser() && service.freeNumUseUp();
        if (dontPass) {
            return AjaxResult.error("普通用户免费识别次数已使用完！");
        } else {
            return AjaxResult.error("会员已到期！");
        }
    }
}

// todo:调用识别接口

```

终于满足3个角色的检查了，加了3层if判断。以后再出现新的角色怎么办？如果功能交给同事来升级，原来的代码轻易不敢动只能再嵌套。


梳理以上需求，3个角色有任意一个通过就可以了。实际上检查时可以按以下先后顺序逐个过，最后一个不满足才返回提示。



```
a. 是否有管理员角色，否进入下一级
b. 是否有VIP角色且未到期，否进入下一级
c. 是否有普通用户角色且满足免费次数条件，否进入下一级；如果没有下一级则检查不通过。

```

#### 重新设计


1. 审批角色接口，主要两个功能：a. 角色判断(当前用户是否为本角色)，b. 是否检查(审批)通过



```
public interface IAudit {
    /**
     * 角色判断：是否为我的责任
     *
     * @return
     */
    boolean isMyDuty();
    
    /**
     * 是否通过
     *
     * @return
     */
    boolean auditPass();
    
    /**
     * 检查(审批)意见：不通过时返回空字符串
     *
     * @return
     */
    String auditMessage();
}

```

2. 审批角色抽象类，实现审批角色接口，并且是3个角色实现类的父类，充当审批角色接口和角色实现类的中间过度。作用是判断检查(审批)是否通过，这里不大容易理解，实际3个角色的实现类分别实现接口就可以了，没有这个中间过度也可以的。为什么要加这个中间类？因为最终检查是否通过要调用isMyDuty和auditPass两个方法，这里可以把这两个方法的调用合并为一个方法，其实就是把判断角色和角色的检查条件统一在这个类而不是在3个实现类里去分别写了，为什么？因为3个实现类要写的判断都是完全一样的代码`isMyDuty()&&auditPass()`，作用就是本来要写3行，现在只写1行。看上去没有必要？因为现在只有3个类呀，如果以后扩展到5个角色，5类那多了。还有，如果是功能修改呢，那就要6个类里分别改了。每改一个类都需要针对这个类单独测试。修改测试花时间多了，这里只有一次修改测试。



```
public abstract class AbstractAudit implements IAudit {
    /**
     * 角色是否检查通过
     *
     * @return
     */
    public boolean checkPass() {
        return isMyDuty() && auditPass();
    }
}

```

3. 3个角色的实现类。


* 管理员:



```
@Service
public class AdminAudit extends AbstractAudit {
    @Autowired
    private IdentifyService identifyService;
    
    @Override
    public boolean isMyDuty() {
        return identifyService.isAdmin();
    }
    
    @Override
    public boolean auditPass() {
        return true;
    }
    
    /**
     * 管理员是没有限制的，所以没有提示
     *
     * @return
     */
    @Override
    public String auditMessage() {
        return "";
    }
}

```

* VIP用户：



```
@Service
public class VipUserAudit extends AbstractAudit {
    @Autowired
    private IdentifyService identifyService;
    
    @Override
    public boolean isMyDuty() {
        return identifyService.isVipUser();
    }
    
    @Override
    public boolean auditPass() {
        return !identifyService.vipUserExpire();
    }
    
    /**
     * 这里还需要优化，因为isMyDuty和auditPass可能被调用两次，可以将isMyDuty、auditPass返回值存在临时变量中
     *
     * @return
     */
    @Override
    public String auditMessage() {
        if (!isMyDuty()) {
            return "不是会员";
        } else if (!auditPass()) {
            return "会员过期";
        }
        return "";
    }
}

```

* 普通用户：



```
@Service
public class NormalUserAudit extends AbstractAudit {
    @Autowired
    private IdentifyService identifyService;
    
    @Override
    public boolean isMyDuty() {
        return identifyService.isNormalUser();
    }
    
    @Override
    public boolean auditPass() {
        return !identifyService.freeNumUseUp();
    }
    
    @Override
    public String auditMessage() {
        return "普通用户免费识别次数已使用完";
    }
}

```

4. 审批责任链类。作用为添加审批人、审批返回结果。



```
public class AuditChain {

    private List<AbstractAudit> chain = new ArrayList<>();

    /**
     * 添加审批人
     *
     * @param auditor
     */
    public void add(AbstractAudit auditor) {
        chain.add(auditor);
    }


    /**
     * 检查/审批
     *
     * @return
     */
    public Result audit() {
        Result result = new Result();
        // 是否检查通过
        boolean pass = chain.stream().anyMatch(a -> a.checkPass());
        result.setPass(pass);
        if (!pass) {
            String msg = chain.stream().map(c -> c.auditMessage()).filter(m -> Strings.isNotBlank(m)).collect(Collectors.joining(","));
            result.setMsg(msg);
        }
        return result;
    }


    @Data
    public class Result {
        private boolean pass;
        private String msg;
    }
}

```

5. 实现检查



```
// 审批责任链中加入3个角色,这里用的Spring Boot开发，3个角色都是容器注入的，其它框架中手动创建实例

// 添加审批人角色
auditChain.add(adminAudit);
auditChain.add(vipUserAudit);
auditChain.add(normalUserAudit);

// 审批结果
AuditChain.Result auditResult = auditChain.audit();
if (!auditResult.isPass()) {
    return AjaxResult.error(auditResult.getMsg());
}

```

#### 总结


最终的实现代码简洁明了，易维护、易扩展升级：


1. 核心方法只有auditChain.add和auditChain.audit，一眼看去就能明白作用是加入审批人和实现审批。
2. 如何扩展功能加入其它角色？创建新的角色类并继承AbstractAudit，并加入到责任链中。不需要在原来的if中嵌套了。
3. 现在的检查是多个角色中有任意一个通过即可，转换到审批场景就是多角色审批，其中一个角色审批通过即可。如果要需求改成多个角色全部审批通过才行呢？其实就是责任人链中or的关系改为and关系。 只需要修改AuditChain类的audit方法，将`chain.stream().anyMatch`改为`chain.stream().allMatch`。anyMatch表示任意一个匹配，allMatch表示全部匹配。如果要在改造前的代码中要实现or到and的变化，原有代码几乎要完全重写。


学习交流：
![](https://img2024.cnblogs.com/blog/574644/202411/574644-20241127164331633-2001342238.jpg)


 本博客参考[wgetcloud全球加速器服务](https://wgetcloud6.org)。转载请注明出处！
