import dns.message
import dns.resolver
from dnslib import DNSRecord, QTYPE

# قم بتعريف دالة للتصفية وفقًا للمعايير المطلوبة
def filter_dns(query):
    blocked_domains = ["example.com", "example.org"]  # قم بتعريف العناوين المحظورة هنا

    # قم بإجراء التحقق من العنوان المستهدف
    if query.q.qname in blocked_domains:
        # إعداد رد فارغ لرفض الطلب
        response = DNSRecord.parse(query.pack())
        response.header.qr = 1
        response.header.ra = 1
        response.header.ancount = 0
        response.rr = []
        return response.pack()

    # إذا لم يكن العنوان محظورًا، قم بتمرير الطلب إلى خادم DNS الأصلي
    resolver = dns.resolver.Resolver()
    upstream_response = resolver.query(str(query.q.qname), query.q.qtype)
    
    # إعداد الرد من الخادم الأصلي
    response = DNSRecord()
    for rdata in upstream_response:
        response.add_answer(*rdata.to_rdataset().to_rdata())

    return response.pack()


# إنشاء خادم DNS
def dns_server():
    from dnslib.server import DNSServer, DNSHandler, BaseResolver

    class DNSFilterResolver(BaseResolver):
        def resolve(self, request, handler):
            query = DNSRecord.parse(request)
            response = filter_dns(query)
            return response

    class DNSFilterHandler(DNSHandler):
        def handle(self):
            request = self.request[0].message
            response = self.server.resolver.resolve(request, self)
            self.request[0].send(response)

    resolver = DNSFilterResolver()
    dns_server = DNSServer(resolver, port=53, address="0.0.0.0")
    dns_server.add_handler(DNSFilterHandler)

    try:
        dns_server.start()
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    dns_server()
