---
layout: post
title: "JNA: Working with Unions"
redirect_from: "/jna-working-with-unions/"
date: 2010-04-02 17:23:29
tags: [jna, java, active directory, win32]
comments: true
dblog_post_id: 93
---

#### The Grey Rat

There’s a giant grey rat outside of my window and a bunch of non-union workers laboring in the rain. The union guy who’s guarding the rat and giving away flyers got too cold and went inside the building. But I digress, the post is about working with Unions in [Java Native Access (JNA).](https://github.com/twall/jna/)

#### Preamble

I was trying to retrieve Active Directory forest trust information via [DsGetForestTrustInformationW](https://learn.microsoft.com/en-us/windows/win32/api/dsgetdc/nf-dsgetdc-dsgetforesttrustinformationw). The function takes a pointer to a [PLSA_FOREST_TRUST_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/ntsecapi/ns-ntsecapi-lsa_forest_trust_information), a pointer to a pointer to an [LSA_FOREST_TRUST_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/ntsecapi/ns-ntsecapi-lsa_forest_trust_information) structure. So far so good, we just need to pay attention to the several levels of indirection: whenever we want the value of a pointer to something, it’s a `ByReference`.

{% highlight java %}
public int DsGetForestTrustInformation(String serverName, String trustedDomainName, int Flags,
        PLSA_FOREST_TRUST_INFORMATION.ByReference ForestTrustInfo);
{% endhighlight %}

{% highlight java %}
public static class PLSA_FOREST_TRUST_INFORMATION extends Structure {
    public static class ByReference extends PLSA_FOREST_TRUST_INFORMATION
        implements Structure.ByReference {
    }
    public LSA_FOREST_TRUST_INFORMATION.ByReference fti;
}
{% endhighlight %}

[LSA_FOREST_TRUST_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/ntsecapi/ns-ntsecapi-lsa_forest_trust_information) is a structure that contains a `RecordCount` number of [PLSA_FOREST_TRUST_RECORD](https://learn.microsoft.com/en-us/windows/win32/api/ntsecapi/ns-ntsecapi-lsa_forest_trust_record) items. Those are pointers, so `Entries` is an array of pointers. Since we want the value of a pointer, we use `ByReference` again.

{% highlight java %}
public static class LSA_FOREST_TRUST_INFORMATION extends Structure {
    public static class ByReference extends LSA_FOREST_TRUST_INFORMATION
        implements Structure.ByReference {
    }

    public NativeLong RecordCount;
    public PLSA_FOREST_TRUST_RECORD.ByReference Entries;
    public PLSA_FOREST_TRUST_RECORD[] getEntries() {
        return (PLSA_FOREST_TRUST_RECORD[]) Entries.toArray(RecordCount.intValue());
    }
}
{% endhighlight %}

A pointer to a record is simply a structure that contains a pointer to the record.

{% highlight java %}
public static class PLSA_FOREST_TRUST_RECORD extends Structure {
    public static class ByReference extends PLSA_FOREST_TRUST_RECORD
        implements Structure.ByReference {
    }
    public LSA_FOREST_TRUST_RECORD.ByReference tr;
}
{% endhighlight %}

#### Union inside a Structure?

Still with me? The record is declared like this:

{% highlight java %}
typedef struct _LSA_FOREST_TRUST_RECORD {
    ULONG Flags;
    LSA_FOREST_TRUST_RECORD_TYPE ForestTrustType; // type of record
    LARGE_INTEGER Time;
    union { // actual data
        LSA_UNICODE_STRING TopLevelName;
        LSA_FOREST_TRUST_DOMAIN_INFO DomainInfo;
        LSA_FOREST_TRUST_BINARY_DATA Data; // used for unrecognized types
    } ForestTrustData;
} LSA_FOREST_TRUST_RECORD;
{% endhighlight %}

Note that MSDN has a mistake [here](https://learn.microsoft.com/en-us/windows/win32/api/ntsecapi/ns-ntsecapi-lsa_forest_trust_record), missing the `Time` field, which gave me lots of headache and wasted hours of my time. Got to use definitions in platform SDK.

This is a union. How do you declare this in JNA?

A union is just like a structure, except that every field lives at an offset zero. In JNA, you must tell the union which field to use before reading the value.

{% highlight java %}
public static class LSA_FOREST_TRUST_RECORD extends Structure {
    public static class ByReference extends LSA_FOREST_TRUST_RECORD
        implements Structure.ByReference {
    }
    public static class UNION extends Union {
        public static class ByReference extends UNION
            implements Structure.ByReference {
        }
        public LSA_UNICODE_STRING TopLevelName;
        public LSA_FOREST_TRUST_DOMAIN_INFO DomainInfo;
        public LSA_FOREST_TRUST_BINARY_DATA Data;
    }
    public NativeLong Flags;
    public int ForestTrustType;
    public LARGE_INTEGER Time;
    public UNION u;
    public void read() {
        super.read();
        switch(ForestTrustType) {
        case NTSecApi.ForestTrustTopLevelName:
        case NTSecApi.ForestTrustTopLevelNameEx:
            u.setType(LSA_UNICODE_STRING.class);
            break;
        case NTSecApi.ForestTrustDomainInfo:
            u.setType(LSA_FOREST_TRUST_DOMAIN_INFO.class);
            break;
        default:
            u.setType(LSA_FOREST_TRUST_BINARY_DATA.class);
            break;
        }
        u.read();
    }
}
{% endhighlight %}

In our case we override `read()` and set the type depending on the `ForestTrustType` value. Then re-read the union from memory. Voila.

#### Notes

Committed to JNA under `com.sun.jna.platform.win32`.

