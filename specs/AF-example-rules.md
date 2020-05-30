
<!-- Section A.6 --> <a id='SA.6'></a>

## Appendix F: Example voting rules

Here we give a set of voting rules for the fields described in our initial
VoteDocuments.

    {
      meta: {
         voting-delay: { op: "Mode", tie_low:false,
                           type:["tuple","uint","uint"] },
         voting-interval: { op: "Median", type:"uint" },
         snip-lifespan: {op: "Mode", type:["tuple","uint","uint","uint"] },
         c-param-lifetime: {op: "Mode", type:["tuple","uint","uint","uint"] },
         s-param-lifetime: {op: "Mode", type:["tuple","uint","uint","uint"] },
         cur-shared-rand: {op: "Mode", min_count: "qfield",
                             type:["tuple","uint","bstr"]},
         prev-shared-rand: {op: "Mode", min_count: "qfield",
                             type:["tuple","uint","bstr"]},
      client-params: {
         recommend-versions: {op:"SetJoin", min_count:"qfield",type:"tstr"},
         require-protos: {op:"BitThreshold", min_count:"sqauth"},
         recommend-protos: {op:"BitThreshold", min_count:"qauth"},
         params: {op:"MapJoin",key_min_count:"qauth",
                     keytype:"tstr",
                     item_op:{op:"Median",min_vote:"qauth",type:"uint"},
                     },
         certs: {op:"SetJoin",min_count:1, type: 'bstr'},
      },
      ; Use same value for server-params.
      relay: {
         meta: {
            desc: {op:"Mode", min_count:"qauth",tie_low:false,
                   type:["uint","bstr"] },
            flags: {op:"MapJoin", key_type:"tstr",
                    item_op:{op:"Mode",type:"bool"}},
            bw: {op:"Median", type:"uint" },
            mbw :{op:"Median", type:"uint" },
            rsa-id: {op:"Mode", type:"bstr"},
        },
        snip: {
           ; ed25519 key is handled as any other value.
           0: { op:"DerivedFrom", fields:[["RM","desc"]],
                 rule:{op:"Mode",type="bstr"} },

           ; ntor onion key.
           1: { op:"DerivedFrom", fields:[["RM","desc"]],
                 rule:{op:"Mode",type="bstr"} },

           ; link specifiers.
           2: { op: "CborDerived",
                 item-op: { op:"DerivedFrom", fields:[["RM","desc"]],
                            rule:{op:"Mode",type="bstr" } } },

           ; software description.
           3: { op:"DerivedFrom", fields:[["RM","desc"]],
                 rule:{op:"Mode",type=["tuple", "tstr", "tstr"] } },

           ; protovers.
           4: { op: "CborDerived",
                 item-op: { op:"DerivedFrom", fields:[["RM","desc"]],
                          rule:{op:"Mode",type="bstr" } } },

           ; families.
           5: { op:"SetJoin", min_count:"qfield", type:"bstr" },

           ; countrycode
           6: { op:"Mode", type="tstr" } ,

           ; 7: exitpolicy.
           7: { op: "CborDerived",
                 item-op: { op: "DerivedFrom", fields:[["RM","desc"],["CP","port-classes"]],
                          rule:{op:"Mode",type="bstr" } } },
        },
        legacy: {
          "sha1-desc": { op:"DerivedFrom",
                          fields:[["RM","desc"]],
                          rule:{op:"Mode",type="bstr"} },
          "mds": { op:"DerivedFrom",
                    fields:[["RM":"desc"]],
                    rule: { op:"ThresholdOp", min_count: "qauth",
                             multi_low:false,
                             type:["tuple", "uint", "uint",
                                   "bstr", "bstr" ] }},
        }
      }
      indices: {
         ; See appendix G.
      }
    }
